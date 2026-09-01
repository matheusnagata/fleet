# Project - Fleet - Upgrading SLES OS on Rancher Downstream Cluster Nodes via SUC

> [!CAUTION]
> This repository is built for **learning and experimentation**. 
> While some configurations may function as expected, they are not guaranteed to meet production standards for security, stability, or performance. Use in production environments at your own risk.

## Project Architecture and Workflow

* **Rancher Fleet Integration:** Fleet orchestrates deployments by pulling configurations from the Git repository and applying them to the `cattle-system` namespace on target downstream clusters.
* **System Upgrade Controller (SUC):** The SUC detects the applied `Plan` resources. In common scenarios, it executes a workflow involving node preparation, workload eviction, and host-level script execution.

## File Reference and Configuration Details

* **`fleet.yaml`:** Defines the deployment scope. The `matchLabels` block controls which clusters receive the OS upgrade. It explicitly excludes the `local` Rancher management cluster to prevent resource exhaustion during mass state reconciliation.
* **`server-plan.yaml`:** Orchestrates the upgrade sequence for control-plane nodes. The `concurrency: 1` parameter enforces strict one-by-one execution to maintain database quorum. The `prepare` block halts the upgrade if any nodes report a `NotReady` status. It integrates a Longhorn data integrity guardrail that explicitly validates Kubernetes API exit codes and iterates through volume robustness states, halting the upgrade if any volumes report a `degraded` or `faulted` status.
* **`agent-plan.yaml`:** Manages the upgrade sequence for worker nodes. The `jobActiveDeadlineSecs: 0` parameter prevents job timeouts during extended control-plane updates. The `prepare` block executes a fail-closed API reachability check, halts execution until all control-plane nodes match the target completion hash, and executes the strict Longhorn volume validation loop before proceeding.
* **`os-upgrade-secret.yaml`:** Stores the execution shell script. It evaluates `zypper -n up` output. It triggers an `nsenter` reboot if core packages update, or exits cleanly if no packages update to prevent infinite loops.

## Execution Observations

* **API Validation & Data Integrity Holds:** Pods transitioning to incoming nodes will purposefully hold in the `Init:0/2` state if Longhorn replicas are actively syncing. The script captures the `kubectl` exit status to prevent fail-open vulnerabilities during API timeouts. It counts volume states natively without relying on external string-matching utilities, explicitly blocking the node from draining until the cluster storage registers `0` degraded volumes.
* **Job States:** When a node triggers the `nsenter` reboot, the active pod abruptly terminates, causing the Kubernetes API to report the pod status as `Unknown`.
* **Recovery Sequencing:** The controller generates a new pod upon node reconnection. The anti-loop logic detects the host is updated, exits with code `0`, and allows the controller to uncordon the node.
* **Artifact Cleanup:** The Kubernetes TTL controller automatically deletes completed jobs and unassociated pods after 15 minutes.
* **Update Triggers:** When initiating a new OS upgrade, operators must modify the `version:` field in both `server-plan.yaml` and `agent-plan.yaml`. This forces the SUC to calculate a new execution hash.

## Standardized Rollback Procedure (SLES OS Patching)

⚠️ **IMPORTANT NOTICE:** Rollback Procedure hasn't yet been tested, pending validation.

In common scenarios, SLES environments utilize the Btrfs filesystem and the `snapper` utility to manage system-level snapshots and recovery. If a deployed patch introduces kernel instability, execute this procedure to revert the target node to its pre-upgrade state.

**Phase 1: Halt the Automated Controller**
Before altering the node operating system, prevent the System Upgrade Controller from attempting to re-apply the patch immediately after the rollback.

1. **Delete the Active Plans:** Execute this on the target downstream cluster to remove the upgrade definitions.
```bash
kubectl delete plan os-server-plan os-agent-plan -n cattle-system

```


2. **Uncordon the Node:** Ensure the Kubernetes scheduler can utilize the node after it reboots and stabilizes.
```bash
kubectl uncordon <node-name>

```



**Phase 2: Execute the OS Rollback**
Access the unstable node directly via SSH or the Rancher Node Shell to perform the filesystem rollback.

1. **List System Snapshots:** SLES automatically generates pre-installation and post-installation snapshots during `zypper` operations.
```bash
snapper list

```


2. **Identify the Target Snapshot:** Locate the snapshot number created immediately prior to the upgrade execution. This is typically labeled "pre" and associated with a `zypp` operation.
3. **Initiate the Rollback:** Execute the rollback command using the identified snapshot number.
```bash
snapper rollback <snapshot-number>

```


4. **Reboot the Node:** Restart the host system to boot into the restored filesystem snapshot.
```bash
reboot

```



**Phase 3: Clear Kubernetes State Markers**
After the node reconnects to the cluster API, remove the upgrade completion labels. This ensures the System Upgrade Controller registers the node as a clean target for future patches.

1. **Remove Plan Labels:**
```bash
kubectl label node <node-name> plan.upgrade.cattle.io/os-server-plan- plan.upgrade.cattle.io/os-agent-plan-

```



**Useful Commands for Validation and Maintenance**

* **Node Reboot Validation:** Verify the node successfully restarted and loaded the new kernel by comparing the Boot ID or Kernel Version.
```bash
kubectl describe node <NODE> | grep -E "Kernel Version|Boot ID"

```


* **Clear Stranded Pods:** Manually remove `Unknown` pods generated by the abrupt `nsenter` reboot sequence.
```bash
kubectl delete pods -n cattle-system --field-selector status.phase=Unknown

```


* **Clear Completed Upgrade Jobs:** Manually remove successful jobs from the namespace prior to automatic garbage collection.
```bash
kubectl delete jobs -n cattle-system -l upgrade.cattle.io/controller=system-upgrade-controller

```