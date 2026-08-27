## ⚠️ IMPORTANT NOTICE: Lab Use Only ⚠️

**This repository and the configurations described within are intended for personal homelab, learning, and experimentation purposes ONLY. This setup is NOT designed, configured, or secured for production environments. Using these configurations in a production setting is strongly discouraged and is done at your own risk.**

### Project Architecture and Workflow

- **Rancher Fleet Integration:** Fleet orchestrates deployments by pulling configurations from the Git repository and applying them to the `cattle-system` namespace on target downstream clusters.

- **System Upgrade Controller (SUC):** The SUC detects the applied `Plan` resources. In common scenarios, it executes a workflow involving node preparation, workload eviction, and host-level script execution.

### File Reference and Configuration Details

- **`fleet.yaml`:** Defines the deployment scope. The `matchLabels` block controls which clusters receive the OS upgrade. Replace empty brackets `{}` with key-value pairs to target specific clusters.

- **`server-plan.yaml`:** Orchestrates the upgrade sequence for control-plane nodes. The `concurrency: 1` parameter enforces strict one-by-one execution to maintain database quorum. The `prepare` block halts the upgrade if any nodes report a `NotReady` status.

- **`agent-plan.yaml`:** Manages the upgrade sequence for worker nodes. It limits concurrency to `1`. The `prepare` block halts execution until all control-plane nodes report the exact completion hash of the server plan.

- **`os-upgrade-secret.yaml`:** Stores the execution shell script. It evaluates `zypper -n up` output. It triggers an `nsenter` reboot if core packages update, or exits cleanly if no packages update to prevent infinite loops.

### Parameters to Modify per Update Cycle

When initiating a new OS upgrade across the infrastructure, modify the following parameters in the Git repository:

1. **Plan Versions (`version` string):** Modify the `version:` field in both `server-plan.yaml` and `agent-plan.yaml`. This change forces the SUC to calculate a new execution hash, triggering the new deployment cycle. Both plans must utilize identical strings.

2. **Cluster Targeting (`fleet.yaml`):** Verify the `clusterSelector` matches the intended target environment. In common scenarios, operators route updates to development clusters prior to production environments by adjusting the `matchLabels` definition.

### Execution Observations

- **Job States:** When a node triggers the `nsenter` reboot, the active pod abruptly terminates, causing the Kubernetes API to report the pod status as `Unknown`.

- **Recovery Sequencing:** The controller generates a new pod upon node reconnection. The anti-loop logic detects the host is updated, exits with code `0`, and allows the controller to uncordon the node.

- **Worker Blockers:** If worker node jobs time out while waiting for the control plane, delete the failed jobs. The controller generates new jobs to continue the sequence.

---

### Standardized Rollback Procedure (SLES OS Patching)

#### ⚠️ IMPORTANT NOTICE: Rollback Procedure hasn't yet been tested, pending validation.

In common scenarios, SLES environments utilize the Btrfs filesystem and the `snapper` utility to manage system-level snapshots and recovery. If a deployed patch introduces kernel instability, execute this procedure to revert the target node to its pre-upgrade state.

**Phase 1: Halt the Automated Controller**

Before altering the node operating system, you must prevent the System Upgrade Controller from attempting to re-apply the patch immediately after the rollback.

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

---

### Useful Commands for Validation

```bash
# Command to validate that the node rebooted when comparing the previous boot ID, same goes for the Kernel Version:
kubectl describe node <NODE> | grep -E "Kernel Version|Boot ID"
```