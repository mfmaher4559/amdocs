
# ADS / Usage DB Resiliency Test Plan

## Objective

Validate that the Cassandra/DSE Usage database can continue serving production-like workload while performing:

1. **Full datacenter rebuild**
2. **Single node replacement**

Both tests must run while active write/read workload is applied to the cluster.

---

# Baseline Required Before Any Failure Test

Before starting either test, capture a clean baseline for at least **30–60 minutes**.

## Baseline workload

Run the normal seeding/client workload against the active cluster.

Capture:

| Metric                    | Baseline Target                 |
| ------------------------- | ------------------------------- |
| Write TPS                 | Current steady-state TPS        |
| Read TPS                  | Current steady-state TPS        |
| P95 write latency         | Baseline value                  |
| P99 write latency         | Baseline value                  |
| P95 read latency          | Baseline value                  |
| P99 read latency          | Baseline value                  |
| Client timeout/error rate | 0 or near 0                     |
| Dropped mutations         | 0                               |
| Pending compactions       | Stable                          |
| Hints on disk             | 0 before test                   |
| CPU                       | Stable, no sustained saturation |
| Disk utilization          | Below safe threshold            |
| Network throughput        | Stable                          |

The failure-mode doc says the deployment uses multi-region / multi-AZ redundancy, RF=3 per region, and `LOCAL_QUORUM` for reads/writes, which is the assumption for these tests. 

---

# Test 1 — Rebuild a Datacenter While Workload Is Running

## Purpose

Validate that one surviving datacenter can continue handling active workload while the failed/passive datacenter is removed, recreated, and rebuilt from the surviving datacenter.

The DC rebuild doc states this procedure removes the failed DC, deletes its resources and data, then adds it back and streams data from the surviving DC. 

---

## Preconditions

| Check         | Requirement                                    |
| ------------- | ---------------------------------------------- |
| Surviving DC  | All nodes `UN`                                 |
| Workload      | Running before failure begins                  |
| Replication   | RF=3 per DC before test                        |
| Consistency   | `LOCAL_QUORUM`                                 |
| Disk headroom | Enough for streaming/compaction                |
| Backups       | Recent backup exists                           |
| Hints         | Near 0 before test                             |
| Monitoring    | Grafana / Mission Control dashboards available |

---

## Test Steps

### Phase 1 — Start workload

Start production-like workload.

Record:

```bash
kubectl get jobs -n hd-soak
kubectl logs -l job-name=<workload-job> -n hd-soak --tail=100 --prefix=true
```

Capture app-side:

```text
TPS
latency p50/p95/p99
timeout count
error count
```

---

### Phase 2 — Simulate failed DC

Remove replication to failed DC:

```sql
ALTER KEYSPACE ads
WITH replication = {'class': 'NetworkTopologyStrategy', 'region-a': '3'};
```

Then remove the failed DC from the `MissionControlCluster` manifest.

Expected result:

```bash
kubectl get CassandraDatacenter,pvc,sts,pod -n <namespace>
```

In failed DC namespace, expected:

```text
No resources found
```

The rebuild procedure expects failed DC CassandraDatacenter, statefulsets, PVCs, and pods to be removed. 

---

### Phase 3 — Confirm surviving DC remains healthy

Run:

```bash
kubectl exec -n <namespace> -it <surviving-pod> -c cassandra -- nodetool status
```

Expected:

```text
All surviving DC nodes = UN
Failed DC no longer present
```

Workload should continue.

Benchmark checkpoint:

| Metric             | Pass Criteria           |
| ------------------ | ----------------------- |
| Writes continue    | Yes                     |
| Reads continue     | Yes                     |
| Client error rate  | No sustained increase   |
| P99 write latency  | Within agreed threshold |
| Dropped mutations  | 0                       |
| Surviving DC nodes | All `UN`                |

---

### Phase 4 — Re-add failed DC

Add the DC back into the `MissionControlCluster`.

Expected:

```bash
kubectl get CassandraDatacenter,pvc,sts,pod -n <namespace>
```

You should see:

```text
CassandraDatacenter created
PVCs Bound
StatefulSets Ready
Pods Running
```

The sample procedure notes that in one 9-node test cluster, all pods reached Running in about **16 minutes**. 

Benchmark:

| Measurement                   | Capture                         |
| ----------------------------- | ------------------------------- |
| Time to recreate DC resources | Start apply → all pods Running  |
| Time for all PVCs Bound       | Start apply → Bound             |
| Time for all pods Ready       | Start apply → Ready             |
| Workload impact               | TPS/latency during pod creation |

---

### Phase 5 — Repair system keyspaces

Run primary-range repairs on surviving DC:

```bash
for ks in system_auth dse_leases system_distributed dse_security dse_system
do
  for pod in $(kubectl get pods -n <namespace> -l cassandra.datastax.com/datacenter=<surviving-dc> -o jsonpath='{.items[*].metadata.name}')
  do
    kubectl exec -n <namespace> "$pod" -c cassandra -- nodetool repair -pr "$ks"
  done
done
```

The DC rebuild procedure specifically calls out repairing `system_auth`, `dse_leases`, `system_distributed`, `dse_security`, and `dse_system`; in the documented 9-node test this took about **15 minutes**. 

---

### Phase 6 — Restore replication to both DCs

```sql
ALTER KEYSPACE ads
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'region-a': '3',
  'region-b': '3'
};
```

Expected:

```text
New writes begin flowing to rebuilt DC
```

The rebuild doc notes that once RF includes the rebuilt DC, new write transactions begin flowing into it. 

---

### Phase 7 — Start DC rebuild task

Example:

```yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: K8ssandraTask
metadata:
  name: region-b-data-rebuild-task
  namespace: hd-soak
spec:
  cluster:
    name: density
    namespace: hd-soak
  dcConcurrencyPolicy: Forbid
  datacenters:
    - region-b
  template:
    concurrencyPolicy: Allow
    maxConcurrentPods: 2
    jobs:
      - name: rebuild-keyspace
        command: rebuild
        args:
          source_datacenter: region-a
          keyspace_name: ads
        restartPolicy: Never
```

The doc recommends `maxConcurrentPods: 2` in the example and says the value should be calculated as `ceil(rackSize / 2)` for rebuilding each rack in two steps. 

---

### Phase 8 — Monitor rebuild

```bash
kubectl describe K8ssandraTask -n hd-soak region-b-data-rebuild-task
```

Per-node streaming:

```bash
kubectl exec -n hd-soak -it <rebuilt-dc-pod> -c cassandra -- nodetool netstats
```

The docs recommend `nodetool netstats` to track streaming percentage and streamed GB per source endpoint. 

---

## DC Rebuild Benchmark Targets

| Benchmark                   | Target / Acceptance                                 |
| --------------------------- | --------------------------------------------------- |
| Surviving DC availability   | No outage                                           |
| Workload continuity         | Writes and reads continue                           |
| Client timeout rate         | No sustained breach                                 |
| P99 write latency           | Define acceptable multiplier, example ≤ 2x baseline |
| P99 read latency            | Define acceptable multiplier, example ≤ 2x baseline |
| DC resource recreation time | Record actual                                       |
| System keyspace repair time | Record actual                                       |
| Rebuild streaming duration  | Record actual                                       |
| Rebuilt DC final status     | All nodes `UN`                                      |
| Hints after recovery        | Return to 0                                         |
| Data validation             | Row counts / query checks match expected            |

---

# Test 2 — Replace a Node While Workload Is Running

## Purpose

Validate that a single failed node can be replaced while workload continues at `LOCAL_QUORUM`.

The failure-mode doc states the system can tolerate a single node outage without affecting availability, with remaining pods continuing to serve reads and writes. 

---

## Preconditions

| Check       | Requirement             |
| ----------- | ----------------------- |
| Cluster     | All nodes `UN`          |
| Workload    | Running                 |
| RF          | 3 per DC                |
| CL          | `LOCAL_QUORUM`          |
| Hints       | 0 or near 0             |
| Target node | Selected and documented |
| Disk/CPU    | Healthy before test     |

---

## Test Steps

### Phase 1 — Start workload and baseline

Capture:

```bash
kubectl get pods -n hd-soak -o wide
kubectl exec -n hd-soak -it <pod> -c cassandra -- nodetool status
```

---

### Phase 2 — Initiate node replacement

Example from the replacement procedure:

```yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: replace-node
  namespace: hd-soak
spec:
  concurrencyPolicy: Forbid
  datacenter:
    name: region-a
    namespace: hd-soak
  jobs:
    - name: replace-node
      command: replacenode
      args:
        pod_name: density-ops-region-a-rack1-sts-0
        maxConcurrentPods: 1
```

The node replacement doc shows `CassandraTask` with `command: replacenode`, `pod_name`, and `maxConcurrentPods: 1`. 

---

### Phase 3 — Monitor replacement

```bash
kubectl describe CassandraTask -n hd-soak replace-node
```

Streaming:

```bash
kubectl exec -n hd-soak -it <replacement-pod> -c cassandra -- nodetool netstats
```

Cluster status:

```bash
kubectl exec -n hd-soak -it <healthy-pod> -c cassandra -- nodetool status
```

Hints:

```bash
kubectl exec -n hd-soak -it <healthy-pod> -c cassandra -- nodetool listendpointspendinghints
```

The hints doc says pending hints can be checked with `nodetool listendpointspendinghints`, and after replay completes the node should report no pending hints. 

---

## Node Replacement Benchmark Targets

| Benchmark               | Target / Acceptance             |
| ----------------------- | ------------------------------- |
| Workload availability   | No outage                       |
| Writes continue         | Yes                             |
| Reads continue          | Yes                             |
| Client timeout rate     | No sustained breach             |
| P99 write latency       | Example ≤ 2x baseline           |
| P99 read latency        | Example ≤ 2x baseline           |
| Replacement task status | Completed                       |
| Final node status       | `UN`                            |
| Hints                   | Return to 0                     |
| Streaming               | Completes successfully          |
| Data validation         | Queries return expected results |

---

# Optional Test 3 — Rack/AZ Replacement While Workload Runs

This is worth including because your docs call out AZ/rack behavior.

The rack replacement doc warns that replacing more than one node in parallel in a 3-node rack greatly increased write latency and degraded write TPS, so for a 3-node rack use `maxConcurrentPods: 1`. 

Example:

```yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: replace-rack
  namespace: hd-soak
spec:
  concurrencyPolicy: Forbid
  datacenter:
    name: region-a
    namespace: hd-soak
  jobs:
    - name: replace-rack
      command: replacenode
      args:
        rack: rack1
        maxConcurrentPods: 1
```

---

# Recommended Benchmark Summary Table

| Test                          | Workload Running? | Primary Metric         | Pass Criteria                                                 |
| ----------------------------- | ----------------: | ---------------------- | ------------------------------------------------------------- |
| Single node replacement       |               Yes | Availability + latency | No outage, node returns `UN`, hints drain                     |
| Rack replacement              |               Yes | TPS degradation        | No outage, acceptable latency/TPS degradation                 |
| Full DC rebuild               |               Yes | Surviving DC stability | Surviving DC remains healthy, rebuilt DC streams successfully |
| DC rebuild with writes active |               Yes | Data consistency       | New writes reach rebuilt DC after RF restore                  |
| Hint replay validation        |               Yes | Pending hints          | Hints return to zero                                          |
| Streaming pressure            |               Yes | Network + latency      | Streaming completes without client failure                    |

---

# Commands to Capture During Every Test

```bash
kubectl get pods -n hd-soak -o wide
kubectl get jobs -n hd-soak
kubectl get cassandradatacenter -n hd-soak
kubectl get cassandratask,k8ssandratask -n hd-soak
```

```bash
nodetool status
nodetool netstats
nodetool tpstats
nodetool compactionstats
nodetool listendpointspendinghints
```

```sql
CONSISTENCY LOCAL_QUORUM;

SELECT COUNT(*) FROM ads.<small_validation_table>;

SELECT *
FROM ads.<known_query_table>
WHERE <known_partition_key_filters>
LIMIT 10;
```

---

# Suggested Acceptance Criteria

Use this as the customer-facing standard:

| Area         | Pass Criteria                                                               |
| ------------ | --------------------------------------------------------------------------- |
| Availability | No complete application outage                                              |
| Consistency  | `LOCAL_QUORUM` reads/writes continue                                        |
| Recovery     | Failed/replaced nodes return to `UN`                                        |
| Hints        | Pending hints return to zero                                                |
| Streaming    | Rebuild/replacement completes without manual repair beyond documented steps |
| Performance  | P95/P99 latency remains within agreed threshold                             |
| Workload     | No sustained failure/error rate                                             |
| Data         | Row counts and sampled queries validate successfully                        |

---

# My Suggested Benchmark Thresholds

For first test run, I would use these:

| Metric              |                 Warning |                                Fail |
| ------------------- | ----------------------: | ----------------------------------: |
| P95 write latency   |           > 2x baseline |            > 3x baseline for 15 min |
| P99 write latency   |         > 2.5x baseline |            > 4x baseline for 15 min |
| Client errors       |                  > 0.5% |                      > 2% sustained |
| Dropped mutations   |            Any increase |                  Sustained increase |
| Pending compactions |    Growing for > 30 min |                    Unbounded growth |
| Hints               | Expected during failure | Fail if not draining after recovery |
| Node status         |   Temporary DN expected |     Fail if not `UN` after recovery |
| Disk usage          |           > 75% warning |                          > 85% fail |
