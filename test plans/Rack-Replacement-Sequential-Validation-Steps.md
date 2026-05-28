# Rack Replacement — Sequential Mode: Validation Steps

## Overview

This document validates Cassandra rack replacement behavior using CassandraTask with sequential node replacement (maxConcurrentPods: 1).

**Key Behavior:** When replacing an entire rack sequentially, nodes are replaced one at a time to minimize cluster impact. Each node completes its bootstrap before the next node begins replacement. This approach maintains cluster stability and prevents excessive write latency degradation.

## Test Environment
- **Namespace:** `hd-soak`
- **Target Rack:** `rack1` in `region-a`
- **Cluster:** density-ops-region-a
- **Scenario:** Full rack replacement with sequential node processing
- **Configuration:** `maxConcurrentPods: 1`

## IMPORTANT: Understanding Sequential Rack Replacement

**Critical Notes:**

### Why Sequential Replacement?

Testing under BAU (Business As Usual) traffic showed that replacing more than one node in parallel in a 3-node rack causes:
- Greatly increased write latency
- Write TPS degradation
- Potential cluster instability

**Best Practice:** The number of concurrently replaced nodes should not be greater than one-third of the nodes in a rack.

### Mission Control UI Limitation

**Note:** The Mission Control UI only supports replacing a single node. Whole-rack replacement must be driven via CassandraTask as shown in this test plan.

### IP Assignment Behavior

Since your Kubernetes environment assigns new IPs from the CNI IP pool on pod restart:
- Each replaced node will receive a **new IP address**
- This follows the "new IP path" behavior
- Nodes will accept writes immediately after joining
- Bootstrap streaming occurs while serving writes

## Prerequisites

- [ ] Kubernetes cluster access configured
- [ ] kubectl CLI installed and configured
- [ ] Access to namespace: `hd-soak`
- [ ] NoSQLBench workload files available in `jobs/` directory
- [ ] Target rack identified: `rack1` in datacenter `region-a`
- [ ] Verify cluster is healthy (all nodes UN)
- [ ] Document current rack topology:
  ```bash
  kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
    nodetool status | grep rack1
  ```

---

## Phase 1: Pre-Replacement Validation

**Objective:** Establish baseline metrics and verify cluster health

### Task 1.1: Document Current Rack State

**Steps:**

1. [ ] List all pods in target rack:
   ```bash
   kubectl get pods -n hd-soak -l cassandra.datastax.com/rack=rack1 \
     -o custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase
   ```

2. [ ] Document current node IPs and Host IDs:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1
   ```

3. [ ] Check data distribution across rack:
   ```bash
   for i in 0 1; do
     echo "=== density-ops-region-a-rack1-sts-$i ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-$i -c cassandra -- \
       nodetool info | grep "Load"
   done
   ```

4. [ ] Check for pending hints:
   ```bash
   # Check if hinted handoff is running
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool statushandoff
   
   # List endpoints with pending hints (if any)
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool listendpointspendinghints
   ```
   
   **Note:** `statushandoff` shows if handoff is running. `listendpointspendinghints` shows which endpoints have pending hints. An empty result means no pending hints.

5. [ ] Document baseline performance metrics:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tablestats | grep -A 5 "Write Count\|Read Count\|Write Latency\|Read Latency"
   ```

**Success Criteria:**
- All rack1 nodes showing UN status
- No pending hints
- Baseline metrics documented
- Node IPs and Host IDs recorded

---

### Task 1.2: Prepare Test Workload

**Steps:**

1. [ ] Create baseline data if not already present:
   ```bash
   kubectl apply -f jobs/nosqlbench-workload-configmap.yaml
   kubectl apply -f jobs/nosqlbench-write-job.yaml
   ```

2. [ ] Wait for baseline data load completion:
   ```bash
   kubectl get jobs -n hd-soak -w
   ```

3. [ ] Verify data is distributed across all nodes:
   ```bash
   # Get credentials from secret
   CASS_USER=$(kubectl get secret density-ops-superuser -n hd-soak -o jsonpath='{.data.username}' | base64 -d)
   CASS_PASS=$(kubectl get secret density-ops-superuser -n hd-soak -o jsonpath='{.data.password}' | base64 -d)
   
   # Query with authentication
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u "$CASS_USER" -p "$CASS_PASS" -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```
   
   **Note:** If you know the credentials, you can use them directly:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```

4. [ ] Clean up the job:
   ```bash
   kubectl delete job nosqlbench-write-test -n hd-soak
   ```

**Success Criteria:**
- Baseline data loaded successfully
- Data queryable from all nodes
- Test workload ready for continuous operation

---

## Phase 2: Initiate Rack Replacement

**Objective:** Launch CassandraTask for sequential rack replacement

### Task 2.1: Create CassandraTask for Rack Replacement

**Steps:**

1. [ ] Create the CassandraTask manifest:
   ```bash
   cat > rack-replacement-task.yaml <<EOF
   apiVersion: control.k8ssandra.io/v1alpha1
   kind: CassandraTask
   metadata:
     name: replace-rack1
     namespace: hd-soak
   spec:
     concurrencyPolicy: Forbid
     datacenter:
       name: region-a
       namespace: hd-soak
     jobs:
       - name: replace-rack1
         command: replacenode
         args:
           rack: rack1
         maxConcurrentPods: 1
   EOF
   ```

2. [ ] Review the task configuration:
   ```bash
   cat rack-replacement-task.yaml
   ```

3. [ ] Apply the CassandraTask:
   ```bash
   kubectl apply -f rack-replacement-task.yaml
   ```

4. [ ] Verify task was created:
   ```bash
   kubectl get cassandratask replace-rack1 -n hd-soak
   ```

**Success Criteria:**
- CassandraTask created successfully
- Task shows in Running status
- maxConcurrentPods set to 1 (sequential)

---

### Task 2.2: Start Continuous Workload

**Objective:** Generate BAU traffic during rack replacement to measure impact

**Steps:**

1. [ ] Start continuous write workload:
   ```bash
   kubectl apply -f jobs/nosqlbench-hint-generation-job.yaml
   ```

2. [ ] Verify workload is running:
   ```bash
   kubectl get jobs -n hd-soak
   kubectl logs -f job/nosqlbench-hint-generation -n hd-soak
   ```

3. [ ] In a separate terminal, monitor write latency:
   ```bash
   while true; do
     echo "=== $(date) ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
       nodetool tablestats baselines.test_data | grep "Write Latency"
     sleep 30
   done
   ```

**Success Criteria:**
- Continuous workload running
- Baseline write latency established
- Monitoring in place

---

## Phase 3: Monitor Sequential Replacement

**Objective:** Validate that nodes are replaced one at a time and track progress

### Task 3.1: Monitor CassandraTask Progress

**Steps:**

1. [ ] Watch CassandraTask status:
   ```bash
   watch -n 5 'kubectl describe cassandratask replace-rack1 -n hd-soak | tail -50'
   ```

2. [ ] Monitor pod statuses in the task:
   ```bash
   kubectl describe cassandratask replace-rack1 -n hd-soak | grep -A 30 "Pod Statuses"
   ```

3. [ ] Verify only one node is being replaced at a time:
   ```bash
   kubectl get pods -n hd-soak -l cassandra.datastax.com/rack=rack1 -w
   ```

4. [ ] Document replacement order and timing:
   ```bash
   kubectl describe cassandratask replace-rack1 -n hd-soak | \
     grep -E "Start Time|Completion Time|Status:"
   ```

**Expected Output Example:**
```
Status:
  Conditions:
    Last Transition Time:  2026-04-29T15:16:41Z
    Message:               
    Reason:                Running
    Status:                True
    Type:                  Running
  Pod Statuses:
    db-dc1-rack1-sts-0:
      Completion Time:  2026-04-28T15:05:56Z
      Start Time:       2026-04-28T05:15:41Z
      Status:           COMPLETED
    db-dc1-rack1-sts-1:
      Completion Time:  2026-04-29T00:31:16Z
      Start Time:       2026-04-28T15:06:01Z
      Status:           COMPLETED
    db-dc1-rack1-sts-2:
      Start Time:  2026-04-29T00:31:21Z
      Status:      RUNNING
  Start Time:    2026-04-28T05:15:41Z
  Succeeded:     2
```

**Success Criteria:**
- ✅ Only one node shows "RUNNING" status at a time
- ✅ Previous nodes show "COMPLETED" before next starts
- ✅ Sequential order maintained (sts-0 → sts-1 → sts-2)
- ✅ Start times show no overlap

---

### Task 3.2: Monitor Individual Node Replacement

**Objective:** Track bootstrap progress for each node being replaced

**Steps:**

1. [ ] Identify the currently replacing node:
   ```bash
   kubectl describe cassandratask replace-rack1 -n hd-soak | \
     grep -A 3 "Status:.*RUNNING"
   ```

2. [ ] Monitor streaming progress on the replacing node:
   ```bash
   REPLACING_NODE=$(kubectl describe cassandratask replace-rack1 -n hd-soak | \
     grep -B 1 "Status:.*RUNNING" | head -1 | awk '{print $1}' | tr -d ':')
   
   kubectl exec -it -n hd-soak $REPLACING_NODE -c cassandra -- \
     nodetool netstats
   ```

3. [ ] Watch streaming progress with formatted output:
   ```bash
   kubectl exec -it -n hd-soak $REPLACING_NODE -c cassandra -- \
     nodetool netstats | awk '/\a+\-\[0-9\]\{1,3\}\.\{3\}/{print}'
   ```

4. [ ] Monitor node status changes:
   ```bash
   while true; do
     echo "=== $(date) ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
       nodetool status 2>/dev/null | grep rack1
     sleep 10
   done
   ```

5. [ ] Check for the new IP assignment:
   ```bash
   kubectl get pod $REPLACING_NODE -n hd-soak -o jsonpath='{.status.podIP}'
   ```

**Expected Streaming Output Example:**
```
/10.1.8.127 : 0.0186096%     0.358676/1927.37GB
/10.1.8.164 : 0.00512277%    0.111914/2184.64GB
/10.1.8.173 : 0.01235903%    0.236621/1914.56GB
/10.1.8.191 : 0.02319468%    0.413935/1784.62GB
/10.1.8.204 : 0.0127195%     0.248485/1953.58GB
/10.1.8.233 : 0.0185481%     0.243199/1311.18GB
```

**Success Criteria:**
- ✅ Streaming progress visible and advancing
- ✅ Multiple source nodes streaming data
- ✅ Percentages increasing over time
- ✅ Node receives new IP address
- ✅ Node eventually reaches UN status

---

### Task 3.3: Validate Write Acceptance During Bootstrap

**Objective:** Confirm replacing node accepts writes immediately (new IP path behavior)

**Steps:**

1. [ ] Wait for node to join the ring (UJ or UN status):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep $REPLACING_NODE
   ```

2. [ ] Attempt direct write to the replacing node:
   ```bash
   kubectl exec -it -n hd-soak $REPLACING_NODE -c cassandra -- \
     cqlsh -e "INSERT INTO baselines.test_data (id, value) VALUES (uuid(), 'test-during-bootstrap');"
   ```
   
   **Expected:** Should succeed (new IP path behavior)

3. [ ] Verify write was accepted:
   ```bash
   kubectl exec -it -n hd-soak $REPLACING_NODE -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Write Count"
   ```

4. [ ] Monitor write latency during bootstrap:
   ```bash
   kubectl exec -it -n hd-soak $REPLACING_NODE -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Write Latency"
   ```

**Success Criteria:**
- ✅ Writes succeed during bootstrap (new IP path)
- ✅ Write count increases on replacing node
- ✅ Write latency remains acceptable
- ✅ No write rejections or timeouts

---

### Task 3.4: Monitor Cluster-Wide Impact

**Objective:** Measure performance degradation during sequential replacement

**Steps:**

1. [ ] Monitor write latency across all nodes:
   ```bash
   for pod in density-ops-region-a-rack1-sts-0 density-ops-region-a-rack1-sts-1 density-ops-region-a-rack2-sts-0; do
     echo "=== $pod ==="
     kubectl exec -it -n hd-soak $pod -c cassandra -- \
       nodetool tablestats baselines.test_data 2>/dev/null | grep "Write Latency"
   done
   ```

2. [ ] Check for dropped mutations:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tpstats | grep -i "dropped"
   ```

3. [ ] Monitor thread pool stats:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tpstats | grep -E "MutationStage|ReadStage"
   ```

4. [ ] Check continuous workload performance:
   ```bash
   kubectl logs job/nosqlbench-hint-generation -n hd-soak | tail -20
   ```

5. [ ] Document peak latency during replacement:
   ```bash
   # Record baseline vs. during-replacement latency
   echo "Baseline: [document from Task 1.1]"
   echo "During replacement: [current measurement]"
   ```

**Success Criteria:**
- ✅ Write latency increase is acceptable (< 2x baseline)
- ✅ No significant dropped mutations
- ✅ Thread pools not saturated
- ✅ Continuous workload continues successfully
- ✅ TPS degradation is minimal

---

## Phase 4: Validate Replacement Completion

**Objective:** Confirm all nodes replaced successfully and cluster is healthy

### Task 4.1: Wait for All Nodes to Complete

**Steps:**

1. [ ] Monitor until all nodes show COMPLETED:
   ```bash
   watch -n 10 'kubectl describe cassandratask replace-rack1 -n hd-soak | grep -A 20 "Pod Statuses"'
   ```

2. [ ] Verify final task status:
   ```bash
   kubectl get cassandratask replace-rack1 -n hd-soak -o yaml | grep -A 10 "status:"
   ```

3. [ ] Document total replacement time:
   ```bash
   kubectl describe cassandratask replace-rack1 -n hd-soak | \
     grep -E "Start Time:|Completion Time:" | head -2
   ```

4. [ ] Verify all rack1 nodes are UN:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1
   ```

**Success Criteria:**
- ✅ All three nodes show "COMPLETED" status
- ✅ CassandraTask shows overall success
- ✅ All rack1 nodes showing UN status
- ✅ Total time documented

---

### Task 4.2: Verify New IP Assignments

**Steps:**

1. [ ] List new IPs for all replaced nodes:
   ```bash
   kubectl get pods -n hd-soak -l cassandra.datastax.com/rack=rack1 \
     -o custom-columns=NAME:.metadata.name,IP:.status.podIP
   ```

2. [ ] Compare with original IPs (from Task 1.1):
   ```bash
   echo "Original IPs: [from Task 1.1]"
   echo "New IPs: [current output]"
   ```

3. [ ] Verify new IPs in nodetool status:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1
   ```

4. [ ] Confirm Host IDs remained the same:
   ```bash
   # Compare Host IDs from Task 1.1 with current
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1 | awk '{print $7}'
   ```

**Success Criteria:**
- ✅ All nodes have new IP addresses
- ✅ Host IDs match original (PVC retained)
- ✅ No duplicate IP entries in cluster
- ✅ All old IPs removed from gossip

---

### Task 4.3: Validate Data Consistency

**Steps:**

1. [ ] Check data distribution across replaced rack:
   ```bash
   for i in 0 1 2; do
     echo "=== density-ops-region-a-rack1-sts-$i ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-$i -c cassandra -- \
       nodetool info | grep "Load"
   done
   ```

2. [ ] Verify row counts match across replicas:
   ```bash
   for pod in density-ops-region-a-rack1-sts-0 density-ops-region-a-rack1-sts-1 density-ops-region-a-rack2-sts-0; do
     echo "=== $pod ==="
     kubectl exec -it -n hd-soak $pod -c cassandra -- \
       cqlsh -e "SELECT COUNT(*) FROM baselines.test_data;" 2>/dev/null
   done
   ```

3. [ ] Run consistency check with CL=ALL:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -e "CONSISTENCY ALL; SELECT COUNT(*) FROM baselines.test_data;"
   ```

4. [ ] Verify no pending hints:
   ```bash
   for pod in density-ops-region-a-rack1-sts-0 density-ops-region-a-rack1-sts-1 density-ops-region-a-rack1-sts-2; do
     echo "=== $pod ==="
     kubectl exec -it -n hd-soak $pod -c cassandra -- \
       nodetool statushandoff 2>/dev/null
   done
   ```

**Success Criteria:**
- ✅ Data load balanced across rack
- ✅ Row counts consistent across replicas
- ✅ CL=ALL queries succeed
- ✅ No pending hints

---

### Task 4.4: Validate Post-Replacement Performance

**Steps:**

1. [ ] Measure current write latency:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Write Latency"
   ```

2. [ ] Compare with baseline (from Task 1.1):
   ```bash
   echo "Baseline: [from Task 1.1]"
   echo "Post-replacement: [current measurement]"
   ```

3. [ ] Run read workload to verify read performance:
   ```bash
   kubectl apply -f jobs/nosqlbench-read-job.yaml
   kubectl logs -f job/nosqlbench-read-test -n hd-soak
   ```

4. [ ] Check for any errors or warnings:
   ```bash
   for pod in density-ops-region-a-rack1-sts-0 density-ops-region-a-rack1-sts-1 density-ops-region-a-rack1-sts-2; do
     echo "=== $pod ==="
     kubectl logs $pod -n hd-soak -c cassandra --tail=50 | grep -i "error\|warn"
   done
   ```

**Success Criteria:**
- ✅ Write latency returned to baseline
- ✅ Read workload completes successfully
- ✅ No errors in logs
- ✅ Performance metrics normal

---

## Phase 5: Optional Repair and Validation

**Objective:** Run repair to ensure anti-entropy and validate final consistency

### Task 5.1: Run Repair on Replaced Rack

**Steps:**

1. [ ] Run repair on each replaced node:
   ```bash
   for i in 0 1 2; do
     echo "=== Repairing density-ops-region-a-rack1-sts-$i ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-$i -c cassandra -- \
       nodetool repair baselines
   done
   ```

2. [ ] Monitor repair progress:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool compactionstats
   ```

3. [ ] Verify repair completion:
   ```bash
   kubectl logs density-ops-region-a-rack1-sts-0 -n hd-soak -c cassandra | \
     grep -i "repair.*complete"
   ```

**Success Criteria:**
- ✅ Repair completes on all nodes
- ✅ No major repair deltas
- ✅ Cluster converges normally

---

## Test Summary

### Critical Validations
- [ ] ✅ **Sequential Replacement:** Only one node replaced at a time
- [ ] ✅ **New IP Path:** Each node received new IP address
- [ ] ✅ **Write Acceptance:** Nodes accepted writes during bootstrap
- [ ] ✅ **Minimal Impact:** Write latency increase acceptable
- [ ] ✅ **Data Consistency:** All replicas consistent after replacement
- [ ] ✅ **Performance Recovery:** Metrics returned to baseline

### Key Metrics Documented
- [ ] Original node IPs and Host IDs
- [ ] New node IPs (post-replacement)
- [ ] Baseline write latency
- [ ] Peak write latency during replacement
- [ ] Total replacement time per node
- [ ] Total rack replacement time
- [ ] Data load per node (before/after)

### Overall Success Criteria
- [ ] All three rack1 nodes replaced successfully
- [ ] Replacement occurred sequentially (one at a time)
- [ ] Each node received new IP address
- [ ] Nodes accepted writes during bootstrap (new IP path)
- [ ] Write latency increase was acceptable (< 2x baseline)
- [ ] No dropped mutations or timeouts
- [ ] All nodes reached UN status
- [ ] Data consistency verified across replicas
- [ ] Performance returned to baseline after completion
- [ ] Continuous workload ran successfully throughout

---

## Troubleshooting

### Issue: Multiple Nodes Replacing Simultaneously

**This indicates configuration error**

**Debug Steps:**
1. Verify maxConcurrentPods is set to 1
2. Check CassandraTask spec
3. Verify Mission Control version (>= 1.18)
4. Review task status for timing overlap

### Issue: Node Stuck in Bootstrap

**Debug Steps:**
1. Check streaming progress: `nodetool netstats`
2. Verify network connectivity between nodes
3. Check source node resources (CPU, memory, disk I/O)
4. Review logs for streaming errors
5. Consider restart if truly stuck (> 24 hours with no progress)

### Issue: Excessive Write Latency

**Expected during replacement, but should be manageable**

**Debug Steps:**
1. Verify only one node is being replaced
2. Check thread pool stats: `nodetool tpstats`
3. Monitor GC activity
4. Check for dropped mutations
5. If excessive (> 3x baseline), consider pausing replacement

### Issue: Data Inconsistency After Replacement

**Debug Steps:**
1. Check for pending hints: `nodetool statushandoff`
2. Run repair: `nodetool repair baselines`
3. Verify replication factor matches expectations
4. Check for network partitions during replacement
5. Review logs for write failures

### Issue: Old IP Still Showing in Cluster

**Debug Steps:**
1. Use `nodetool assassinate <old-ip>` to remove
2. Verify gossip state: `nodetool gossipinfo`
3. Check for split-brain scenarios
4. Review replacement logs for errors

---

## Cleanup

1. [ ] Stop continuous workload:
   ```bash
   kubectl delete job nosqlbench-hint-generation -n hd-soak
   kubectl delete job nosqlbench-read-test -n hd-soak
   ```

2. [ ] Delete CassandraTask:
   ```bash
   kubectl delete cassandratask replace-rack1 -n hd-soak
   ```

3. [ ] Optionally drop test keyspace:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -e "DROP KEYSPACE IF EXISTS baselines;"
   ```

4. [ ] Verify cluster health:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status
   ```

5. [ ] Clean up test files:
   ```bash
   rm rack-replacement-task.yaml
   ```

---

## References

- Rack Rebuild Test Plan: `test plans/rack-rebuild.pdf`
- Single Node Replacement (New IP): `test plans/Single-Node-Replacement-Validation-Steps.md`
- Single Node Replacement (Reused IP): `test plans/Single-Node-Replacement-Reused-IP-Validation-Steps.md`
- Apache Cassandra Streaming: https://cassandra.apache.org/doc/latest/operating/topo_changes.html

---

## Revision History

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-05-27 | 1.0 | Initial validation steps for sequential rack replacement | Test Team |