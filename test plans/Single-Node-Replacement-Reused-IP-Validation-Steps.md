# Single Node Replacement — Reused IP Path: Validation Steps

## Overview

This document validates Cassandra node replacement behavior when the replacement node retains its original IP address (reused IP path).

**Key Behavior:** When the IP address is reused, the new node is treated as the same node rejoining the cluster. It initiates bootstrap procedure, however it does NOT accept new write transactions. Consistency is achieved by replaying the hints collected in the coordinators. Once the streaming operation is complete, the node accepts client connections and starts participating in all transaction types. All outstanding hints are replayed to the new node.

## Test Environment
- **Namespace:** `hd-soak`
- **Target Pod:** `density-ops-region-a-rack1-sts-0`
- **Cluster:** density-ops-region-a
- **Scenario:** Node restart where IP address is retained

## IMPORTANT: Achieving IP Reuse

**Critical Note:** In your Kubernetes environment, the IP address changes even when using `nodetool stopdaemon` and manual restart. This indicates your environment does NOT support the "reused IP path" scenario as originally designed.

### IP Change Observed:
- Original IP: 10.56.26.15
- After restart: 10.56.26.16

**This means you are actually testing the "new IP path" scenario, not the "reused IP path" scenario.**

### Why IP Reuse is Difficult:

The "reused IP path" scenario requires:
1. Same pod identity (name)
2. Same IP address
3. Same Host ID in Cassandra

In most Kubernetes environments:
- Pod deletion → New IP assignment
- Cassandra restart → May trigger new IP from CNI
- Only certain CNI plugins (Calico with IP pools, etc.) support IP retention

### Alternative: Test with Cassandra Restart Only

To truly test the "reused IP path" behavior, you would need:

1. **Stop Cassandra without killing the pod:**
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     pkill -9 java
   ```
   Then wait for liveness probe to restart Cassandra automatically

2. **Use a CNI that supports IP retention:**
   - Configure Calico with IP address management
   - Use static IP assignments
   - Requires cluster-level configuration

### Recommendation:

Since your environment assigns new IPs, you have two options:

**Option A:** Accept that you're testing the "new IP path" and use the other test plan:
- `test plans/Single-Node-Replacement-Validation-Steps.md`

**Option B:** Focus on the hint replay portion of this test:
- Skip the write rejection validation (requires reused IP)
- Test only hint generation and replay after node returns
- This is still valuable for validating hint mechanism

**Verification:** Always verify the IP address. If it changes, you are testing "new IP path" behavior.

## Prerequisites

- [ ] Kubernetes cluster access configured
- [ ] kubectl CLI installed and configured
- [ ] Access to namespace: `hd-soak`
- [ ] NoSQLBench workload files available in `jobs/` directory
- [ ] Target node identified: `density-ops-region-a-rack1-sts-0`
- [ ] Verify cluster is healthy (all nodes UN)
- [ ] Document current IP address of target pod:
  ```bash
  kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}'
  ```

---

## Phase 1: Validate Write Rejection During Bootstrap

**Objective:** Confirm the replacement node does NOT accept new writes during bootstrap

### Task 1.1: Prepare Test Environment

**Steps:**

1. [ ] Create the NoSQLBench write workload ConfigMap:
   ```bash
   kubectl apply -f jobs/nosqlbench-workload-configmap.yaml
   ```

2. [ ] Verify ConfigMap was created:
   ```bash
   kubectl get configmap simple-write-nb -n hd-soak
   ```

3. [ ] Populate baseline data:
   ```bash
   kubectl apply -f jobs/nosqlbench-write-job.yaml
   ```

4. [ ] Wait for job completion and verify data:
   ```bash
   kubectl get jobs -n hd-soak -w
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```

5. [ ] Clean up the job:
   ```bash
   kubectl delete job nosqlbench-write-test -n hd-soak
   ```

**Success Criteria:**
- ConfigMap created successfully
- 100,000 rows inserted
- Data queryable from all nodes

---

### Task 1.2: Simulate Node Failure and Generate Hints

**Note:** In environments with automatic restart (liveness probes, init containers), Cassandra may restart immediately after `stopdaemon`. This is acceptable - we'll generate hints during the brief downtime and restart period.

**Steps:**

1. [ ] Document current node IP address:
   ```bash
   TARGET_IP=$(kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}')
   echo "Target IP: $TARGET_IP"
   ```

2. [ ] Check current hint status (baseline):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool statushandoff
   ```

3. [ ] In one terminal, monitor node status continuously:
   ```bash
   # macOS/Linux compatible
   while true; do
     clear
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
       nodetool status 2>/dev/null | grep -E "$TARGET_IP|rack1-sts-0|Address"
     sleep 1
   done
   ```

4. [ ] In another terminal, stop Cassandra and immediately start generating writes:
   ```bash
   # Stop Cassandra
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- nodetool stopdaemon
   
   # Immediately start write workload to generate hints
   kubectl apply -f jobs/nosqlbench-hint-generation-job.yaml
   ```

5. [ ] Monitor for brief "DN" status (may be very short if auto-restart is enabled):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep "$TARGET_IP"
   ```
   
   **Note:** If Cassandra restarts immediately, you may see it go from UN → DN → UJ very quickly, or skip DN entirely and go straight to UJ (bootstrap mode).

6. [ ] Watch for node to enter bootstrap mode (UJ status):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep "$TARGET_IP"
   ```

**Success Criteria:**
- Node IP remains the same ($TARGET_IP)
- Node shows "DN" status
- Write workload is running
- Hints are being generated (check with `nodetool statushandoff`)

---

### Task 1.3: Restart Cassandra and Trigger Bootstrap

**Objective:** Manually restart Cassandra to trigger bootstrap with the same IP address

**Steps:**

1. [ ] Verify node is still down and hints are accumulating:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep "$TARGET_IP"
   
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool statushandoff
   ```

2. [ ] Restart Cassandra process inside the container:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     /bin/bash -c "cassandra -R"
   ```
   
   **Note:** The `-R` flag tells Cassandra to start in the foreground. Alternatively, you can restart the pod:
   ```bash
   kubectl delete pod density-ops-region-a-rack1-sts-0 -n hd-soak
   ```
   
   **Warning:** Deleting the pod may result in a new IP address. If the IP changes, you'll be testing the "new IP path" instead.

3. [ ] Monitor for Cassandra startup in logs:
   ```bash
   kubectl logs -f -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra
   ```

4. [ ] Watch for node to enter bootstrap mode (UJ status):
   ```bash
   while true; do
     clear
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
       nodetool status 2>/dev/null | grep -E "$TARGET_IP|Address"
     sleep 2
   done
   ```

5. [ ] Verify IP address (check if it changed):
   ```bash
   NEW_IP=$(kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}')
   echo "Original IP: $TARGET_IP"
   echo "Current IP: $NEW_IP"
   if [ "$TARGET_IP" = "$NEW_IP" ]; then
     echo "✅ IP REUSED - Testing reused IP path"
   else
     echo "⚠️  IP CHANGED - Testing new IP path instead"
   fi
   ```

**Success Criteria:**
- Cassandra process restarts successfully
- IP address checked and documented (may or may not change)
- Node enters bootstrap mode (shows "UJ" status)
- Bootstrap mode confirmed in logs

---

### Task 1.4: Validate Write Rejection During Bootstrap

**CRITICAL TEST:** Confirm writes are rejected while node is bootstrapping

**Steps:**

1. [ ] Attempt direct write to the bootstrapping node:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e \
     "INSERT INTO baselines.test_data (id, value) VALUES (uuid(), 'test-during-bootstrap');"
   ```
   
   **Expected:** Should fail with error or timeout

2. [ ] Try write with explicit consistency level ONE:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e \
     "CONSISTENCY ONE; INSERT INTO baselines.test_data (id, value) VALUES (uuid(), 'test-cl-one');"
   ```
   
   **Expected:** Should fail even with CL=ONE

3. [ ] Verify write through coordinator succeeds (routes to other nodes):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     cqlsh -u <username> -p <password> -e \
     "INSERT INTO baselines.test_data (id, value) VALUES (uuid(), 'test-via-coordinator');"
   ```
   
   **Expected:** Should succeed

4. [ ] Document error messages received from direct write attempts

5. [ ] Verify node is still in bootstrap mode:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

6. [ ] Monitor streaming progress:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool netstats
   ```

**Success Criteria:**
- ✅ Direct writes to bootstrapping node FAIL
- ✅ Error messages indicate node is not ready for writes
- ✅ Writes through coordinators succeed (routed to other replicas)
- ✅ Node remains in "UJ" status during test
- ✅ Streaming progresses normally

---

### Task 1.5: Wait for Bootstrap Completion

**Steps:**

1. [ ] Monitor streaming progress until completion:
   ```bash
   while true; do
     echo "=== $(date) ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
       nodetool netstats | grep -A 20 "Mode:"
     sleep 30
   done
   ```

2. [ ] Wait for node status to change from "UJ" to "UN":
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

3. [ ] Verify bootstrap completion in logs:
   ```bash
   kubectl logs -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra | \
     grep -i "bootstrap.*complete"
   ```

**Success Criteria:**
- Streaming completes successfully (100%)
- Node status changes to "UN" (Up, Normal)
- Bootstrap completion logged

---

## Phase 2: Validate Hint Replay After Bootstrap

**Objective:** Confirm hints from coordinators replay after bootstrap completes

### Task 2.1: Generate Hints During Downtime

**Steps:**

1. [ ] Before restarting the node (while it's still down), generate write traffic to create hints:
   ```bash
   kubectl apply -f jobs/nosqlbench-hint-generation-job.yaml
   ```

2. [ ] Monitor job execution:
   ```bash
   kubectl logs -f job/nosqlbench-hint-generation -n hd-soak
   ```

3. [ ] Wait for job completion

4. [ ] Verify hints are stored on coordinator nodes:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool statushandoff
   
   kubectl exec -it -n hd-soak density-ops-region-a-rack2-sts-0 -c cassandra -- \
     nodetool statushandoff
   ```

5. [ ] Check hint table statistics:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tablestats system.hints | grep -A 10 "Table: hints"
   ```

**Success Criteria:**
- Write job completes successfully
- Hints visible in `nodetool statushandoff`
- Hint count > 0 in system.hints table

---

### Task 2.2: Confirm Node Reaches UN Status

**Steps:**

1. [ ] Verify node status is "UN":
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

2. [ ] Verify no active streaming:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool netstats
   ```

3. [ ] Document bootstrap completion time

**Success Criteria:**
- Status is "UN" (Up, Normal)
- No active streaming sessions
- Bootstrap complete

---

### Task 2.3: Validate Hint Replay Begins

**CRITICAL TEST:** Confirm hints start replaying automatically to the recovered node

**Steps:**

1. [ ] Check hint status immediately after UN status:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool statushandoff
   ```

2. [ ] Monitor hint delivery in thread pool stats:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tpstats | grep HintsDispatcher
   ```

3. [ ] Watch hint count decrease over time:
   ```bash
   while true; do
     echo "=== $(date) ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
       nodetool tablestats system.hints | grep "Number of partitions"
     sleep 10
   done
   ```

4. [ ] Monitor hint delivery logs on coordinator:
   ```bash
   kubectl logs -f -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra | \
     grep -i "hint"
   ```

5. [ ] Monitor hint delivery logs on target node:
   ```bash
   kubectl logs -f -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra | \
     grep -i "hint"
   ```

**Success Criteria:**
- ✅ Hint count decreases over time
- ✅ HintsDispatcher thread pool shows activity
- ✅ Hint delivery logged on coordinator nodes
- ✅ No hint delivery errors

---

### Task 2.4: Validate Hint Replay Completion

**Steps:**

1. [ ] Wait for hint count to reach zero:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool tablestats system.hints | grep "Number of partitions"
   ```

2. [ ] Verify no hints remain on any coordinator:
   ```bash
   for pod in density-ops-region-a-rack1-sts-1 density-ops-region-a-rack2-sts-0 density-ops-region-a-rack3-sts-0; do
     echo "=== Checking $pod ==="
     kubectl exec -it -n hd-soak $pod -c cassandra -- nodetool statushandoff
   done
   ```

3. [ ] Verify data consistency across nodes:
   ```bash
   # Count rows on recovered node
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   
   # Count rows on other nodes
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   
   kubectl exec -it -n hd-soak density-ops-region-a-rack2-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```

4. [ ] Verify no hint delivery failures:
   ```bash
   kubectl logs -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra | \
     grep -i "hint.*fail\|hint.*error"
   ```

**Success Criteria:**
- ✅ Hint count reaches zero on all coordinators
- ✅ `nodetool statushandoff` shows no pending hints
- ✅ Row counts match across all nodes
- ✅ No hint delivery errors in logs
- ✅ Data consistency verified

---

### Task 2.5: Validate Node Accepts Writes After Recovery

**Steps:**

1. [ ] Attempt direct write to recovered node:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e \
     "INSERT INTO baselines.test_data (id, value) VALUES (uuid(), 'test-after-recovery');"
   ```
   
   **Expected:** Should succeed

2. [ ] Run write workload:
   ```bash
   kubectl apply -f jobs/nosqlbench-write-job.yaml
   ```

3. [ ] Monitor write job completion:
   ```bash
   kubectl logs -f job/nosqlbench-write-test -n hd-soak
   ```

4. [ ] Verify writes distributed to recovered node:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Write Count"
   ```

**Success Criteria:**
- ✅ Direct writes succeed
- ✅ Write workload completes successfully
- ✅ Recovered node receives write traffic
- ✅ No write errors or timeouts

---

## Test Summary

### Critical Validations
- [ ] ✅ **Phase 1:** Writes rejected during bootstrap (reused IP behavior confirmed)
- [ ] ✅ **Phase 2:** Hints replayed automatically after bootstrap completion

### Key Metrics Documented
- [ ] Original IP address
- [ ] IP address after restart (should match original)
- [ ] Number of hints generated
- [ ] Bootstrap duration
- [ ] Hint replay duration
- [ ] Write rejection error messages
- [ ] Final data consistency check results

### Overall Success Criteria
- [ ] Node restarted with same IP address (reused IP path confirmed)
- [ ] Node entered bootstrap mode ("UJ" status)
- [ ] Writes rejected during bootstrap
- [ ] Bootstrap streaming completed successfully
- [ ] Node reached "UN" status
- [ ] Hints replayed automatically after bootstrap
- [ ] All hints delivered successfully
- [ ] Data consistency verified across all nodes
- [ ] Node accepts writes after recovery

---

## Troubleshooting

### Issue: Node Accepts Writes During Bootstrap

**This indicates test failure - expected behavior is NOT occurring**

**Debug Steps:**
1. Verify IP address matches original
2. Check if PVC was retained
3. Review bootstrap mode in logs
4. Confirm this is reused IP scenario, not new IP

### Issue: Hints Not Replaying

**Debug Steps:**
1. Check hint age (may have expired if >3 hours)
2. Verify network connectivity
3. Check HintsDispatcher thread pool
4. Force hint delivery: `nodetool handoff <target-ip>`
5. If expired, run repair: `nodetool repair baselines`

### Issue: Bootstrap Stuck

**Debug Steps:**
1. Check streaming progress: `nodetool netstats`
2. Verify network connectivity
3. Check source node resources
4. Consider restart if truly stuck

---

## Cleanup

1. [ ] Delete test jobs:
   ```bash
   kubectl delete job nosqlbench-write-test -n hd-soak
   kubectl delete job nosqlbench-hint-generation -n hd-soak
   ```

2. [ ] Optionally drop test keyspace:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "DROP KEYSPACE IF EXISTS baselines;"
   ```

3. [ ] Verify cluster health:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status
   ```

---

## References

- Rack Rebuild Test Plan: `test plans/rack-rebuild.pdf`
- New IP Path Test Plan: `test plans/Single-Node-Replacement-Validation-Steps.md`
- Apache Cassandra Hints: https://cassandra.apache.org/doc/latest/operating/hints.html

---

## Revision History

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-05-27 | 1.0 | Initial validation steps for reused IP path | Test Team |