# janitor: a failed GPUReset ends with `Complete=True`, so condition-only consumers (including fault-remediation) read it as success

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug, janitor, fault-remediation, api |

## Status on main

**Still present on main@229530ed.** `reconcileTerminalFailure` in `janitor/pkg/controller/gpureset_controller.go`
still sets `Complete` to `metav1.ConditionTrue` with the failure reason; `CRStatusChecker.checkCondition` in
`fault-remediation/pkg/crstatus/checker.go` still maps `True` to `CRStateSucceeded` without looking at the reason
(4ceafbd3, #2093, changed read-error handling only); `docs/configuration/fault-remediation.md` still documents
`completeConditionType: "Complete"` for `GPUReset`; and a failed Job still produces the fixed message
"GPU reset failed".

## Steps to reproduce

1. Create a `GPUReset` whose reset Job will fail (for example: a GPU held by another process, or a `driverRoot`
   where `nvidia-smi` is not found).
2. Wait for it to finish.
3. `kubectl get gpureset <name> -o yaml`: `status.phase: Failed`, but `Complete=True` with reason
   `ResetJobFailed` (or `RestoreTimeoutExceeded`).
4. `kubectl wait --for=condition=Complete gpureset/<name>` returns success.

**Component:**
- Janitor GPUReset controller (`reconcileTerminalFailure`, `reconcileCompletion` in
  `janitor/pkg/controller/gpureset_controller.go`);
- the GPUReset API (`janitor.dgxc.nvidia.com/v1alpha1`);
- fault-remediation's CR status checker (`fault-remediation/pkg/crstatus/checker.go`).

## Environment

Seen on two clusters:

| | Cluster 1 | Cluster 2 |
| --- | --- | --- |
| Hardware | NVIDIA GB300 NVL72 compute trays, driver 595.91.07 | NVIDIA HGX B300 (8 × B300 SXM6 per x86_64 node), driver 595.91.07 |
| Kubernetes | upstream-conformant Kubernetes v1.36 | a managed Kubernetes service |
| Physical reboot step | the `kind` simulator | a cloud provider's out-of-band reset API |

Common to both:
- NVSentinel runs the Janitor and janitor-provider only.
- `GPUReset` objects name one GPU by UUID.
- The reset Job is the Janitor's real reset Job; only the physical reboot step differs between the clusters.

## Actual behaviour (what happened)

Every failed reset we ran ended with `Complete=True`. Four resets, four failures, four `Complete=True`.

**HGX B300, the reset ran and `nvidia-smi` refused it** (the device was held; see issue 02):

```yaml
status:
  phase: Failed
  conditions:
  - {type: ResetJobCompleted, status: "True", reason: ResetJobFailed, message: GPU reset failed}
  - {type: ServicesRestored,  status: "True", reason: ServiceRestoreSucceeded, message: gpu-operator managed services are Ready}
  - {type: Complete,          status: "True", reason: ResetJobFailed, message: GPU reset failed}
```

Two earlier attempts on the same node ended the same way (`Complete=True`, `ResetJobFailed`, `phase: Failed`):
1. The reset image pull used most of the Job's deadline.
2. A `driverRoot` misconfiguration on our side meant `nvidia-smi` was not found.

**GB300 NVL72, the reset Job failed, then the restore timed out:**

```yaml
status:
  phase: Failed
  conditions:
  - {type: ResetJobCompleted, status: "True",  reason: ResetJobFailed,       message: GPU reset failed}
  - {type: ServicesRestored,  status: "False", reason: RestoringServices,    message: Re-deploying gpu-operator managed services}
  - {type: Complete,          status: "True",  reason: RestoreTimeoutExceeded, message: failed to restore gpu-operator managed services within the timeout period}
```

The source behaves the same way. `reconcileTerminalFailure` sets `Complete=True` with the failure reason and
`phase: Failed`; success sets `Complete=True` with `GPUResetSucceeded`. The `Complete` condition therefore means
"terminal", not "succeeded".

`status.phase` is correct (`Failed`), but the condition is what generic tooling reads:
- `kubectl wait --for=condition=Complete` returns success on a failed reset;
- so does any controller that follows the "Complete=True means repaired" convention.

### NVSentinel's own fault-remediation reads it as success

We found this by reading the code; we have not run it, because fault-remediation is disabled on our clusters.

- `CRStatusChecker.checkCondition` maps `<completeConditionType>=True` to `CRStateSucceeded` without looking at the
  reason.
- The reconciler then logs "CR completed successfully, marking same-session equivalent event remediated".
- `docs/configuration/fault-remediation.md` documents a `GPUReset` action with `completeConditionType: "Complete"`.

So a deployment that follows the documented example would record a failed GPU reset as a successful remediation and
would not retry it.

### Two smaller points from the same objects

- **The original cause is lost.** When a failed reset is followed by a restore timeout, `Complete` carries
  `RestoreTimeoutExceeded`, and the reset failure survives only in `ResetJobCompleted`.
- **The cause of the Job failure is not surfaced.** The message is always "GPU reset failed"; the real cause ("In
  use by another client", or `chroot: failed to run command 'nvidia-smi'`) is only in the Job pod's log.

## Expected behaviour

A failed reset should not report the condition a consumer reads as success. Either:
- follow the batch/v1 Job convention: `Complete=True` only on success and a separate `Failed=True` on terminal
  failure; or
- set `Complete=False` with the failure reason. This is how `RebootNode` already reports failure
  (`NodeReady=False`, `Failed` / `Timeout`), and it is the contract the custom-remediation tutorial describes ("False
  to make NVSentinel retry").

## Impact

Anything that reads only the condition status records a reset that did not happen as a repair, and returns a
still-faulty GPU to service. This includes NVSentinel's fault-remediation as documented, `kubectl wait`, and
external orchestrators.

## Suggested fix / design direction

1. On terminal failure, set `Complete=False` with the failure reason, or add a `Failed` condition and keep
   `Complete` for success only. The API is `v1alpha1`, so this is the cheapest time to change it. Note the change in
   the release notes, since existing consumers may test `Complete=True` alone.
2. Until then, make fault-remediation's status checker accept an optional per-action list of success reasons, or
   read `status.phase`. Change the documented `GPUReset` example to use it.
3. Keep the first failure reason in `Complete` when later steps also fail, or record every failure reason in the
   message.
4. Copy the first error line of the reset Job's output into the `ResetJobCompleted` message.

## Workaround we use

Our integration lists success reasons for each action. For `GPUReset` only `Complete=True` with reason
`GPUResetSucceeded` counts as success; `True` with any other reason is a failed handoff (fail closed). `RebootNode`
accepts only `NodeReady=True/Succeeded` and stops waiting at once on `False` with `Failed` or `Timeout`.
