Yes — for the **`ads.usage` table specifically**, the issue is the primary key:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday),
  recordopeningtime,
  eventid
)
```

That means the table is partitioned by **market bucket + bill cycle day**, not by subscriber.

So the table is good for:

```text
Get all usage records for a market / cycle year / cycle month / cycle close day
```

But it is poor for:

```text
Get records for one subscriber
Get records for one very large subscriber
Page through one subscriber’s records
```

Because `subscriberid` is only a normal column, not part of the key. Inside each partition, rows are ordered by:

```sql
recordopeningtime, eventid
```

not by subscriber. So records from many subscribers are mixed together.

The key client-facing point is:

> The `usage` table is currently modeled around the market/cycle extraction use case. It is not modeled around subscriber-level retrieval. Because multiple subscribers share the same partition and `subscriberid` is not in the primary key, large subscribers can dominate a partition and make subscriber-specific retrieval inefficient or impractical.

Recommended direction:

Keep `ads.usage` for the market/cycle extraction path, but add a second table for subscriber reads:

```sql
CREATE TABLE ads.usage_by_subscriber (
    billingmarketid text,
    subscriberid text,
    cycleyear int,
    cyclemonth int,
    cyclecloseday int,
    recordopeningtime timestamp,
    eventid text,
    ...
    PRIMARY KEY (
      (billingmarketid, subscriberid, cycleyear, cyclemonth, cyclecloseday),
      recordopeningtime,
      eventid
    )
);
```

For the very large subscribers, add a bucket:

```sql
PRIMARY KEY (
  (billingmarketid, subscriberid, cycleyear, cyclemonth, cyclecloseday, bucket),
  recordopeningtime,
  eventid
)
```

So the short answer is:

> The `usage` table should remain optimized for market/cycle extraction. It should not be stretched to also serve subscriber-specific reads. Subscriber reads need their own query table, and large subscribers likely need additional bucketing.


--------------------

That makes sense. With a **70 TB `usage` table**, I would **not recommend a second full copy** unless absolutely required.

For this client, I’d position it this way:

> Since `ads.usage` is already very large, duplicating the table for a subscriber access pattern may not be practical. The better near-term recommendation is to adjust the existing `usage` table model so subscriber records are no longer interleaved inside shared market/cycle partitions.

Current key:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday),
  recordopeningtime,
  eventid
)
```

The core problem is that the partition key groups multiple subscribers together, while the clustering key sorts only by `recordopeningtime` and `eventid`. The ticket explicitly says records from different subscribers are interleaved and that large subscribers can prevent retrieval of smaller subscribers in the same partition. 

## Better single-table direction

Change the `usage` table so `subscriberid` participates in the key.

Recommended shape:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday, subscriberbucket),
  subscriberid,
  recordopeningtime,
  eventid
)
```

Where:

```text
subscriberbucket = murmur2hash(subscriberid) % N
```

This keeps the table market/cycle oriented, but adds a subscriber grouping before time ordering.

## Why this helps

Instead of rows being ordered like this inside the partition:

```text
time1 - subscriber A
time1 - subscriber B
time1 - subscriber C
time2 - subscriber A
time2 - subscriber B
```

They become grouped like:

```text
subscriber A - time1
subscriber A - time2
subscriber B - time1
subscriber B - time2
subscriber C - time1
```

That means subscriber reads can page through a cleaner contiguous slice instead of scanning interleaved data.

## For very large subscribers

For subscribers with 93k–186k records per bill cycle, I would consider adding a second bucket based on time:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday, subscriberbucket, timebucket),
  subscriberid,
  recordopeningtime,
  eventid
)
```

Example:

```text
timebucket = day or hour derived from recordopeningtime
```

That prevents a single high-volume subscriber from creating a very large logical slice.

## Tradeoff to explain to customer

This improves subscriber retrieval, but changes the market extraction behavior.

Market extraction would now need to query across all `subscriberbucket` values, for example bucket `0..N-1`. That is usually acceptable if the extraction job is Spark-based or parallelized.

## My recommendation

Do **not** create a second 70 TB table.

Recommend a revised single `usage` table design:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday, subscriberbucket),
  subscriberid,
  recordopeningtime,
  eventid
)
```

And for large-subscriber protection:

```sql
PRIMARY KEY (
  (billingmarketidpartkey, cycleyear, cyclemonth, cyclecloseday, subscriberbucket, timebucket),
  subscriberid,
  recordopeningtime,
  eventid
)
```

Client-facing wording:

> Given the size of the `usage` table, duplicating the data into a second query table may not be desirable. The better option is to remodel the existing `usage` table so subscriber identity participates in the primary key. This prevents subscriber records from being interleaved only by event time and allows subscriber-specific reads to page through a more contiguous range. The tradeoff is that market-level extraction will need to fan out across subscriber buckets, but that is easier to parallelize than scanning mixed partitions containing very large subscribers.
