# Single Node Replacement — New IP Path: Detailed Validation Steps

## Overview
This document provides a step-by-step guide to validate Cassandra node replacement behavior when the replacement node receives a different IP address.

## Test Environment
- **Namespace:** `hd-soak`
- **Target Pod:** `density-ops-region-a-rack1-sts-0`
- **Cluster:** density-ops-region-a

## Expected Behavior Summary
- ✅ Replacement node accepts writes immediately after joining cluster
- ❌ Does NOT serve reads/client traffic initially
- ✅ Bootstrap streaming begins automatically
- ✅ Node joins full traffic participation after bootstrap completes

---

## Prerequisites

### Environment Setup
- [ ] Kubernetes cluster access configured
- [ ] kubectl CLI installed and configured
- [ ] Access to namespace: `hd-soak`
- [ ] CassandraTask CRD available in cluster
- [ ] Monitoring/logging tools ready (optional but recommended)

### Pre-Test Verification
- [ ] Target node identified: `density-ops-region-a-rack1-sts-0`
- [ ] Document current cluster topology
- [ ] Verify cluster is healthy (all nodes UN)
- [ ] Baseline metrics captured (TPS, latency)
- [ ] Verify current pod status:
  ```bash
  kubectl get pods -n hd-soak
  ```

---

## Phase 1: Initiate Node Replacement

### Task 1.1: Initiate Node Replacement
**Objective:** Replace the target node by deleting the pod and letting StatefulSet recreate it

**Note:** K8ssandra/cass-operator does not support a "replace" command in CassandraTask. Node replacement is achieved by deleting the pod, which triggers the StatefulSet controller to create a new pod with a new IP address.

**Steps:**
1. [ ] Verify current pod status and capture baseline information:
   ```bash
   kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o wide
   ```
   
2. [ ] Document the current pod IP address for comparison:
   ```bash
   kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}'
   ```

3. [ ] Check current node status in the ring:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

4. [ ] Delete the target pod to trigger replacement:
   ```bash
   kubectl delete pod density-ops-region-a-rack1-sts-0 -n hd-soak
   ```

5. [ ] Monitor pod recreation:
   ```bash
   kubectl get pods -n hd-soak -w | grep rack1-sts-0
   ```

6. [ ] Wait for pod to reach Running state (all 3/3 containers ready)

**Expected Result:** Pod is deleted and StatefulSet creates a new pod with same name but different IP

**Validation Commands:**
```bash
# Check pod status
kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak

# Verify new IP address (should be different)
kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}'
```

**Success Criteria:**
- Pod successfully deleted
- New pod created by StatefulSet
- Pod reaches Running state (3/3 Ready)
- New pod has different IP address than original
- Pod name remains the same: `density-ops-region-a-rack1-sts-0`

---

### Task 1.2: Confirm Node Joins Ring
**Objective:** Verify the replacement node successfully joins the Cassandra ring

**Steps:**
1. [ ] Wait for new pod to be created (may take 1-2 minutes)
2. [ ] Monitor pod replacement:
   ```bash
   kubectl get pods -n hd-soak -w
   ```
3. [ ] Check node status from an existing node (e.g., rack1-sts-1):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- nodetool status
   ```
4. [ ] Verify new node appears in ring with status "UJ" (Up, Joining)
5. [ ] Document the new node's IP address

**Expected Result:** New node visible in ring with "UJ" status

**Success Criteria:**
- New node listed in `nodetool status` output
- Status shows "UJ" (Up, Joining) initially
- IP address is different from original node (10.x.x.x)

---

## Phase 2: Write Traffic Validation During Bootstrap

### Task 2.0: Prepare NoSQLBench Workload (One-Time Setup)
**Objective:** Set up the NoSQLBench workload ConfigMap before starting the test

**Prerequisites:**
- NoSQLBench workload ConfigMap and Job manifests are in `jobs/` directory
- Files: `jobs/nosqlbench-workload-configmap.yaml` and `jobs/nosqlbench-write-job.yaml`

**Steps:**
1. [ ] Create the NoSQLBench workload ConfigMap (one-time setup):
   ```bash
   kubectl apply -f jobs/nosqlbench-workload-configmap.yaml
   ```
   
   This creates a ConfigMap named `simple-write-nb` with a workload that:
   - Creates keyspace `baselines` with NetworkTopologyStrategy (RF=3 per DC)
   - Creates table `test_data` with simple schema
   - Inserts 100,000 rows during rampup phase

2. [ ] Verify ConfigMap was created:
   ```bash
   kubectl get configmap simple-write-nb -n hd-soak
   ```

**Success Criteria:**
- ConfigMap `simple-write-nb` exists in namespace `hd-soak`

---

### Task 2.1: Delete Target Pod to Trigger Replacement
**Objective:** Initiate node replacement by deleting the target pod

**Steps:**
1. [ ] Verify current pod status and capture baseline:
   ```bash
   kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o wide
   ```

2. [ ] Document the current pod IP address:
   ```bash
   kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}'
   ```

3. [ ] Check current node status in the ring:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- nodetool status 
   ```

4. [ ] Delete the target pod to trigger replacement:
   ```bash
   kubectl delete pod density-ops-region-a-rack1-sts-0 -n hd-soak
   ```

5. [ ] Monitor pod recreation (keep this running in a separate terminal):
   ```bash
   kubectl get pods -n hd-soak -w | grep rack1-sts-0
   ```

6. [ ] Wait for pod to reach Running state (3/3 containers ready)

7. [ ] Verify new IP address (should be different):
   ```bash
   kubectl get pod density-ops-region-a-rack1-sts-0 -n hd-soak -o jsonpath='{.status.podIP}'
   ```

8. [ ] Check node status - should show "UJ" (Up, Joining):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

**Expected Result:** Pod deleted, recreated with new IP, node shows "UJ" status

**Success Criteria:**
- Pod successfully deleted and recreated
- New pod has different IP address
- Node appears in ring with "UJ" (Up, Joining) status
- Pod reaches Running state (3/3 Ready)

---

### Task 2.2: Start Write Traffic During Bootstrap
**Objective:** Generate write traffic while node is bootstrapping to verify writes continue

**IMPORTANT:** Execute this task immediately after the pod starts bootstrapping (UJ status)

**Steps:**
1. [ ] Start the NoSQLBench write workload:
   ```bash
   kubectl apply -f jobs/nosqlbench-write-job.yaml
   ```

2. [ ] Monitor the Job status:
   ```bash
   kubectl get jobs -n hd-soak -w
   ```

4. [ ] Check pod status (wait for ContainerCreating to complete):
   ```bash
   kubectl get pods -n hd-soak | grep nosqlbench
   ```
   
   **Note:** The pod will show "ContainerCreating" while pulling the NoSQLBench image. This may take 1-2 minutes on first run.

5. [ ] Watch the Job logs in real-time (once pod is Running):
   ```bash
   kubectl logs -f job/nosqlbench-write-test -n hd-soak
   ```
   
   If you get "waiting to start: ContainerCreating", wait a minute and try again.

6. [ ] **In a separate terminal**, monitor streaming progress on the bootstrapping node:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool netstats
   ```
   
   Look for:
   - Mode: BOOTSTRAP
   - Streaming progress percentages
   - Source nodes streaming data

7. [ ] Monitor NoSQLBench output for:
   - Progress percentage (0% → 100%)
   - Operations completed (should reach 100,000)
   - Any error messages
   - Completion status

8. [ ] Let the workload run to completion (typically 1-2 minutes)

**Expected Output:**
```
001 (pending,current,complete)=(1,1,0) 0.00%
001 (pending,current,complete)=(0,0,2) 100.00% (last report)
001 (pending,current,complete)=(75114,38,24848) 24.86%
001 (pending,current,complete)=(30028,39,69933) 69.96%
001 (pending,current,complete)=(0,0,100000) 100.00% (last report)
```

**Expected Result:** Writes continue successfully while node is bootstrapping

**Success Criteria:**
- NoSQLBench job starts successfully
- Write operations complete without errors during bootstrap
- No timeout exceptions
- Streaming progresses on the bootstrapping node
- NoSQLBench shows successful operation completion

---

### Task 2.3: Validate Writes Succeeded During Bootstrap
**Objective:** Confirm write operations were successfully committed during bootstrap

**Steps:**
1. [ ] Review NoSQLBench Job completion status:
   ```bash
   kubectl get job nosqlbench-write-test -n hd-soak
   ```
   
   Should show `COMPLETIONS: 1/1`

2. [ ] Review final NoSQLBench output from logs:
   ```bash
   kubectl logs job/nosqlbench-write-test -n hd-soak | tail -20
   ```
   
   Look for:
   - `100.00% (last report)` - indicates completion
   - `(0,0,100000)` - shows all 100,000 operations completed
   - No error messages or exceptions

3. [ ] Query the data to confirm writes persisted:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack2-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```

4. [ ] Verify row count matches expected value (100,000 rows)

5. [ ] Check data across multiple nodes to confirm replication:
   ```bash
   # Check from rack1 (non-bootstrapping node)
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   
   # Check from rack3
   kubectl exec -it -n hd-soak density-ops-region-a-rack3-sts-0 -c cassandra -- \
     cqlsh -u <username> -p <password> -e "SELECT COUNT(*) FROM baselines.test_data;"
   ```

6. [ ] Document write success rate from NoSQLBench output

**Expected Result:** All writes successfully committed to cluster during bootstrap

**Success Criteria:**
- Job shows COMPLETIONS: 1/1
- NoSQLBench logs show 100.00% completion
- All 100,000 operations completed successfully
- Zero error messages in logs
- Data queryable from all coordinator nodes
- Row counts match expected values (100,000)
- Data replicated across all racks (RF=3)
- Writes succeeded while node was in "UJ" (bootstrapping) state

---

## Phase 3: Bootstrap Streaming Completion

### Task 3.1: Monitor Streaming to Completion
**Objective:** Track bootstrap streaming until it completes and node reaches UN status

**Steps:**
1. [ ] Monitor streaming progress continuously until completion:
   ```bash
   while true; do
     echo "=== $(date) ==="
     kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
       nodetool netstats | grep -A 20 "Mode:"
     sleep 60
   done
   ```

2. [ ] Document streaming metrics:
   - Streaming rates (MB/s)
   - Total data transferred (GB)
   - Source nodes streaming data
   - Estimated time to completion

3. [ ] Watch for streaming completion indicators:
   - Progress reaches 100%
   - Mode changes from BOOTSTRAP to NORMAL
   - No active streaming sessions

4. [ ] Verify no streaming failures or retries in logs:
   ```bash
   kubectl logs -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra | grep -i "stream\|bootstrap"
   ```

**Expected Result:** Streaming completes successfully

**Success Criteria:**
- Streaming progresses steadily to 100%
- No stalled streams
- All peer nodes complete successfully
- No streaming errors in logs
- Reasonable streaming duration (depends on data size)

---

### Task 3.2: Confirm Node Reaches UN Status
**Objective:** Verify node completes bootstrap and becomes fully operational

**Steps:**
1. [ ] Monitor node status until bootstrap completes:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```

2. [ ] Wait for status to change from "UJ" to "UN" (Up, Normal)

3. [ ] Verify streaming has completed (netstats shows no active streams):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- nodetool netstats
   ```

4. [ ] Check node logs for bootstrap completion message:
   ```bash
   kubectl logs -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra | grep -i "bootstrap.*complete"
   ```

5. [ ] Verify node owns expected token ranges:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- nodetool ring
   ```

6. [ ] Document total time taken for bootstrap

**Expected Result:** Node status changes to "UN"

**Success Criteria:**
- Status shows "UN" (Up, Normal)
- No active streaming sessions
- Bootstrap completion logged
- Node owns expected token ranges
- Total bootstrap time documented

---

## Phase 4: Read Traffic Validation

### Task 4.1: Verify Initial Read Behavior
**Objective:** Confirm node does NOT serve reads during bootstrap

**Steps:**
1. [ ] While node is still in "UJ" status, attempt direct reads from cqlsh pod:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-cqlsh-5995d64fc4-m4hwh -- \
     cqlsh <new-node-ip> -e "SELECT * FROM keyspace.table LIMIT 10;"
   ```
2. [ ] Verify connection is refused or reads are not served
3. [ ] Check client driver behavior (should route around bootstrapping node)
4. [ ] Document any error messages received

**Expected Result:** Reads are NOT served from bootstrapping node

**Success Criteria:**
- Direct reads to new node fail or are rejected
- Client drivers route reads to other nodes
- No data inconsistencies

---

### Task 4.2: Confirm Node Reaches UN Status
**Objective:** Verify node completes bootstrap and becomes fully operational

**Steps:**
1. [ ] Monitor node status until bootstrap completes:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-1 -c cassandra -- \
     nodetool status | grep rack1-sts-0
   ```
2. [ ] Wait for status to change from "UJ" to "UN" (Up, Normal)
3. [ ] Verify streaming has completed (netstats shows no active streams):
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- nodetool netstats
   ```
4. [ ] Check node logs for bootstrap completion message:
   ```bash
   kubectl logs -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra | grep -i "bootstrap.*complete"
   ```
5. [ ] Document time taken for full bootstrap

**Expected Result:** Node status changes to "UN"

**Success Criteria:**
- Status shows "UN" (Up, Normal)
- No active streaming sessions
- Bootstrap completion logged
- Node owns expected token ranges

---

### Task 4.3: Validate Read Traffic Resumes
**Objective:** Confirm node accepts read traffic after bootstrap completion

**Steps:**
1. [ ] Create the NoSQLBench read workload ConfigMap:
   ```bash
   kubectl apply -f jobs/nosqlbench-read-configmap.yaml
   ```
   
   This creates a ConfigMap named `simple-read-nb` with a workload that:
   - Reads from `baselines.test_data` table
   - Performs 50,000 read operations
   - Uses 10 threads for concurrent reads
   - Converts UUIDs to strings for WHERE clause (fixed codec issue)

2. [ ] Create the NoSQLBench read Job:
   ```bash
   kubectl apply -f jobs/nosqlbench-read-job.yaml
   ```

4. [ ] Check pod status (wait for ContainerCreating to complete):
   ```bash
   kubectl get pods -n hd-soak | grep nosqlbench-read
   ```
   
   **Note:** The pod will show "ContainerCreating" while pulling the NoSQLBench image. Wait for it to reach "Running" status.

5. [ ] Monitor the read Job (once pod is Running):
   ```bash
   kubectl logs -f job/nosqlbench-read-test -n hd-soak
   ```
   
   If you get "waiting to start: ContainerCreating", wait a minute and try again.
   
   **Expected output:**
   ```
   read (pending,current,complete)=(41076,10,8915) 17.83%
   read (pending,current,complete)=(10097,10,39893) 79.80%
   read (pending,current,complete)=(0,0,50000) 100.00% (last report)
   ```

6. [ ] Verify reads complete successfully:
   ```bash
   kubectl get job nosqlbench-read-test -n hd-soak
   ```
   
   Should show `COMPLETIONS: 1/1`

7. [ ] Check that the new node (rack1-sts-0) is receiving read traffic by monitoring its metrics:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Read Count"
   ```

8. [ ] Compare read latency across nodes:
   ```bash
   # Check read latency on the new node
   kubectl exec -it -n hd-soak density-ops-region-a-rack1-sts-0 -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Read Latency"
   
   # Compare with another node
   kubectl exec -it -n hd-soak density-ops-region-a-rack2-sts-0 -c cassandra -- \
     nodetool tablestats baselines.test_data | grep "Read Latency"
   ```

**Expected Result:** New node successfully serves read traffic

**Success Criteria:**
- NoSQLBench read job completes successfully (COMPLETIONS: 1/1)
- All 50,000 read operations complete without errors
- New node shows read count > 0 in tablestats
- Read latency comparable to other nodes
- No consistency issues or timeout errors

---

## Test Summary Checklist

### Overall Success Criteria
- [ ] Pod deleted and recreated with new IP address
- [ ] Node joined ring with "UJ" (Up, Joining) status
- [ ] Writes continued successfully during bootstrap (NoSQLBench: 100,000 operations)
- [ ] Streaming progressed to 100% completion
- [ ] Node reached "UN" (Up, Normal) status
- [ ] Read traffic resumed after bootstrap (NoSQLBench: 50,000 operations)
- [ ] All region-a nodes show "UN" status
- [ ] No data loss or consistency issues

### Documentation Requirements
- [ ] Test execution date/time recorded
- [ ] Original IP address documented
- [ ] New IP address documented
- [ ] Bootstrap duration documented
- [ ] Data size streamed documented
- [ ] NoSQLBench write results captured
- [ ] NoSQLBench read results captured
- [ ] Any issues or anomalies noted
- [ ] Screenshots/logs captured

---

## Troubleshooting Guide

### Issue: Node Stuck in "UJ" Status
**Possible Causes:**
- Streaming errors
- Network connectivity issues
- Insufficient resources

**Debug Steps:**
1. Check streaming errors: `nodetool netstats`
2. Review node logs: `kubectl logs -n ads-db <pod> -c cassandra`
3. Check network connectivity between nodes
4. Verify disk space and resources

---

### Issue: Writes Failing During Bootstrap
**Possible Causes:**
- Insufficient replicas available
- Consistency level too high
- Network issues

**Debug Steps:**
1. Check consistency level configuration
2. Verify RF vs available nodes
3. Review coordinator logs for timeout details
4. Check network latency between nodes

---

### Issue: Streaming Not Progressing
**Possible Causes:**
- Network bandwidth saturation
- Source node overloaded
- Disk I/O bottleneck

**Debug Steps:**
1. Check network utilization
2. Monitor CPU/disk on source nodes
3. Review streaming throughput settings
4. Check for compaction storms

---

### Issue: Pod Running but Cassandra Node Shows DN
**Possible Causes:**
- Cassandra process crashed but container didn't exit
- Gossip protocol failure
- Network connectivity at Cassandra level (not pod level)
- Cassandra stuck in bad state after restart
- JVM issues (GC pause, OOM)
- Liveness/readiness probe not detecting Cassandra failure

**Debug Steps:**
1. Map DN IP to specific pod:
   ```bash
   kubectl get pods -n hd-soak -o wide | grep <down-ip>
   ```

2. Check Cassandra logs:
   ```bash
   kubectl logs -n hd-soak <pod-name> -c cassandra --tail=200
   ```

3. Check if Cassandra is responding:
   ```bash
   kubectl exec -it -n hd-soak <pod-name> -c cassandra -- nodetool status
   ```

4. Check Cassandra process:
   ```bash
   kubectl exec -it -n hd-soak <pod-name> -c cassandra -- ps aux | grep java
   ```

5. Check gossip state:
   ```bash
   kubectl exec -it -n hd-soak <pod-name> -c cassandra -- nodetool gossipinfo
   ```

6. Check for network issues from other nodes:
   ```bash
   kubectl exec -it -n hd-soak density-ops-region-b-rack2-sts-0 -c cassandra -- \
     nc -zv <down-node-ip> 7000
   ```

**Resolution:**
- If Cassandra is stuck: restart the pod
  ```bash
  kubectl delete pod <pod-name> -n hd-soak
  ```
- If it's a gossip issue: may self-resolve or need cluster-wide restart
- If network issue: investigate pod networking
- Document as pre-existing if it was down before test
- Region-a test results remain valid

---

## References
- Main Test Plan: `TP-Rack-Rebuild.md`
- CassandraTask Documentation: [Link to docs]
- Cassandra Bootstrap Process: [Link to Apache docs]

---

## Revision History
| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-05-26 | 1.0 | Initial detailed validation steps | Test Team |