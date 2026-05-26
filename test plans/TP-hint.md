# Hints Monitoring & Delivery Test Plan

## 1. Simulate a Node Outage

Purpose:
Verify hints are created when a replica node becomes unavailable.

Steps:

1. Identify a Cassandra node.
2. Stop Cassandra or isolate the node/network.
3. Continue generating writes against the cluster.
4. Confirm the coordinator begins storing hints.

Validation:
Run:

```bash
nodetool listendpointspendinghints
```

Expected:

* DOWN node appears
* Total hints increases
* Total files increases
* Newest/Oldest timestamps update

Example from doc:

```text
Status: DOWN
Total hints: 3170247
```

---

## 2. Validate Grafana Hint Metrics During Outage

Purpose:
Verify Mission Control Grafana dashboards reflect hint accumulation.

Validation Areas:

* Total Hints / Cluster
* Hints on Disk / Cluster

Expected:

* Metrics increase while node is unavailable
* Disk usage grows

Referenced in document:
Mission Control Cluster Grafana dashboard. 

---

## 3. Restore Node Availability

Purpose:
Verify hint replay begins after node recovery.

Steps:

1. Bring node back online.
2. Wait for gossip/state recovery.

Validation:
Run:

```bash
nodetool listendpointspendinghints
```

Expected:

* Status changes from DOWN → UP
* Pending hints decrease
* Hint file count decreases

Example from doc:

```text
Status: UP
Total hints reduced from 3170247 → 1117007
```

---

## 4. Validate Hint Replay Metrics

Purpose:
Verify replay activity is visible in Grafana.

Validation:
Observe:

* Hint replay rate
* succeeded-hints metric

Expected:

* Replay rate increases during delivery
* succeeded-hints eventually returns to zero after completion

Referenced in document:
“Hints section of the Mission Control Cluster Grafana dashboard shows hint replay rate.” 

---

## 5. Verify Complete Hint Drain

Purpose:
Ensure all hints are eventually delivered.

Validation:
Run:

```bash
nodetool listendpointspendinghints
```

Expected:

```text
This node does not have hints for other endpoints
```

Referenced in document. 

---

## 6. Hint Window Retention Test

Purpose:
Validate hints are retained only within configured outage window.

Document states:

* Default hint window = 3 days

Suggested Test:

1. Keep node offline beyond hint window.
2. Continue writes.
3. Bring node online.

Expected:

* Older hints are discarded
* Full repair may be required afterward

---

## 7. Disk Consumption Test

Purpose:
Validate hints do not exhaust coordinator disk.

Validation:
Monitor:

* Hints on disk
* Filesystem utilization
* Hint file growth rate

Expected:

* Hint growth observable
* Disk remains within operational thresholds

---

## 8. Multi-Node Failure Behavior

Purpose:
Verify hint handling during multiple simultaneous node outages.

Suggested Validation:

* Stop multiple replicas
* Generate writes
* Observe pending hints per endpoint

Expected:

* Separate pending hint tracking per unavailable node

---

## 9. Maintenance Window Recovery Test

Purpose:
Validate expected behavior during planned maintenance.

Steps:

1. Drain/shutdown node for maintenance.
2. Continue workload.
3. Restore node.

Expected:

* Hints replay successfully
* No data inconsistency remains

---

## 10. Post-Replay Consistency Validation

Purpose:
Ensure replay restored consistency.

Suggested Validation:

* Run application reads
* Compare row counts/checksums
* Run repair verification if needed

Expected:

* No missing writes after replay completes
