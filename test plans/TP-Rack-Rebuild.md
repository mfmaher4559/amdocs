# Cassandra Node & Rack Replacement Test Plan

## 1. Single Node Replacement — New IP Path

Purpose:
Validate behavior when replacement node receives a different IP address.

Documented Expected Behavior:

* Replacement node accepts writes immediately after joining cluster
* Does NOT serve reads/client traffic initially
* Bootstrap streaming begins automatically
* Node joins full traffic participation after bootstrap completes

Validation Steps:

1. Replace one node using CassandraTask
2. Confirm node joins ring
3. Generate write traffic during bootstrap
4. Validate writes succeed
5. Confirm streaming activity
6. Confirm node eventually accepts reads

Validation Commands:

```bash id="mnsk5g"
kubectl describe cassandratask replace-node -n ads-db
```

```bash id="0v1yrn"
kubectl exec -it -n ads-db <pod> -c cassandra -- nodetool netstats
```

Success Criteria:

* Writes continue successfully
* Streaming progresses
* Node reaches `UN`
* Read traffic resumes after bootstrap

Referenced behavior: 

---

## 2. Single Node Replacement — Reused IP Path

Purpose:
Validate replacement behavior when original IP is reused.

Documented Expected Behavior:

* Replacement node does NOT accept writes during bootstrap
* Hints replay after bootstrap completion
* Node joins all traffic after replay completes

Validation Steps:

1. Replace node while preserving/reusing IP
2. Generate writes during outage
3. Confirm hints accumulate
4. Validate bootstrap streaming
5. Validate hints replay after bootstrap

Validation Commands:

```bash id="v3f8uk"
nodetool listendpointspendinghints
```

```bash id="4hkp8w"
nodetool netstats
```

Success Criteria:

* Hints accumulate during bootstrap
* Hints replay after bootstrap
* Node reaches normal state
* No data inconsistency

Referenced behavior: 

---

## 3. Whole Rack Replacement — Sequential Mode

Purpose:
Validate full rack replacement using CassandraTask.

Configuration:

```yaml id="7cn1dz"
maxConcurrentPods: 1
```

Validation Steps:

1. Launch replace-rack CassandraTask
2. Verify only one node replaced at a time
3. Monitor bootstrap progress per node
4. Validate cluster remains operational

Success Criteria:

* Nodes replaced sequentially
* No excessive TPS degradation
* Cluster remains healthy

Referenced task example: 

---

## 4. Negative Test — Excessive Parallel Rebuilds

Purpose:
Validate documented performance degradation when replacing too many nodes simultaneously.

Configuration:

```yaml id="xtfxn4"
maxConcurrentPods > 1/3 rack size
```

Example:

* 3-node rack
* set:

```yaml id="0vjlwm"
maxConcurrentPods: 2
```

Validation Steps:

1. Run parallel rebuilds
2. Generate BAU workload
3. Monitor:

   * write latency
   * TPS
   * client timeout behavior

Expected Result:

* Increased write latency
* TPS degradation
* Possible instability under load

Referenced warning: 

---

## 5. Mission Control UI Validation

Purpose:
Validate supported UI functionality.

Validation:

1. Confirm single-node replacement available in UI
2. Execute replacement from UI
3. Confirm task launches correctly
4. Confirm whole-rack replacement is NOT exposed in UI

Expected:

* Single-node replacement supported
* Rack replacement CassandraTask-only

Referenced note: 

---

## 6. Pre-1.18 Compatibility Validation

Purpose:
Validate legacy replacement behavior.

Documented Behavior:

* maxConcurrentPods ignored before MC 1.18
* Replacement always serial

Validation Steps:

1. Run CassandraTask on pre-1.18 cluster
2. Set:

```yaml id="0qv9hh"
maxConcurrentPods: 3
```

3. Observe replacement order

Expected:

* Only one node replaced at a time

Referenced note: 

---

## 7. CassandraTask Progress Monitoring

Purpose:
Validate CassandraTask status reporting.

Validation:

```bash id="lvd1kw"
kubectl describe cassandratask replace-rack -n ads-db
```

Expected:

* Running/completed nodes visible
* Completion timestamps accurate
* PodStatuses update properly

Referenced example: 

---

## 8. nodetool netstats Streaming Validation

Purpose:
Validate rebuild streaming metrics.

Validation:

```bash id="6hgc7d"
kubectl exec -it -n ads-db <pod> -c cassandra -- nodetool netstats
```

Expected:

* Per-peer streaming progress visible
* Percentages advance
* Total GB transferred increases

Referenced example: 

---

# Additional Recommended Validations

## 9. Post-Rebuild Consistency Validation

Purpose:
Ensure no data loss after replacement.

Validation:

* Run application reads
* Compare row counts/checksums
* Validate consistency level behavior

Expected:

* No missing writes
* No stale reads

---

## 10. Repair Validation After Replacement

Purpose:
Ensure anti-entropy consistency restored.

Validation:

```bash id="f2rj2g"
nodetool repair
```

Expected:

* No major repair deltas
* Cluster converges normally

---

## 11. Hint Replay Validation

Purpose:
Ensure coordinators replay stored hints correctly.

Validation:

```bash id="t2rb7t"
nodetool listendpointspendinghints
```

Expected:

* Hint count decreases after node returns
* Eventually:

```text id="35l7qo"
This node does not have hints for other endpoints
```

---

## 12. Client Impact Validation

Purpose:
Measure operational impact during replacement.

Metrics to Monitor:

* P99 read latency
* P99 write latency
* TPS
* dropped mutations
* timeouts
* GC pressure
* network throughput

Expected:

* Acceptable degradation during sequential rebuild
* Significant degradation during negative concurrency test
