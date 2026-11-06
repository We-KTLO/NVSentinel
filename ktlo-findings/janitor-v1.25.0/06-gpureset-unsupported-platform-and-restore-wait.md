# janitor: GPUReset has no pre-flight for platforms whose host cannot reset a single GPU (GB300 NVL72); restore waited for operands the site never deployed

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug / enhancement, janitor, gpu-reset |

## Status on main

**Part A appears fixed by 3c637ad4 (#1989, "DRA support in janitor's GPUReset") on main@229530ed.** The
controller now records, at teardown start, which managed services had a pod on the node
(`status.managedServices.nodeLabels`, `startTearDown` / `runningApps`) and restores and waits only for those
(`managedApps`). An operand the site never deployed is no longer waited for. Verify on a cluster with DCGM and
the DCGM exporter disabled in the GPU Operator before dropping part A; if confirmed, file part B alone.

**Part B is still present on main@229530ed**: there is no reset-capability pre-flight and no
`ResetNotSupported`-style reason in the controller, the API types or `gpu-reset/gpu_reset.sh`.

**Part B must be re-run before filing** (see the note below and "Evidence limits").

## Steps to reproduce

Part A (v1.25.0):
1. GPU Operator with standalone DCGM and the DCGM exporter disabled (host runs its own DCGM).
2. Create a `GPUReset` naming one GPU on a cordoned and drained node.
3. Observe: after the reset Job ends, `ServicesRestored=False/RestoringServices` for 10 minutes, then
   `Complete=True/RestoreTimeoutExceeded`, even though the deployed operands came back.

Part B (to re-run before filing):
1. On a GB300 NVL72 tray, deploy the Janitor with a correct `driverRoot` (`/host` for a host-installed driver).
2. Create a `GPUReset` naming one GPU; keep the reset Job pod's log.
3. Expected platform answer from our separate test: `nvidia-smi -r -i <gpu>` reports `Not Supported`. The Janitor
   still tears down the operands and runs the Job first.

> **Before filing:** part B's claim about GB300 rests on a separate test of ours, not on the Janitor run below (see
> "Evidence limits"). Re-run one `GPUReset` on a GB300 tray with a corrected `driverRoot` and keep the reset Job's
> log, or file part A on its own. Part A now appears fixed on `main` (see "Status on main"); if that is confirmed,
> there is nothing left to file without the part B re-run.

**Component:**
- Janitor GPUReset controller (`janitor/pkg/controller/gpureset_controller.go`, `restoreServices` /
  `checkPodsReady`);
- the `gpu-operator` service manager registry (`janitor/pkg/gpuservices/manager.go`);
- the reset script (`gpu-reset/gpu_reset.sh`).

## Environment

- **Hardware:** NVIDIA GB300 NVL72 compute trays (4 GPUs per tray, Grace host). Driver 595.91.07.
- **Kubernetes:** an upstream-conformant distribution, v1.36. A batch scheduler shares the hosts, so a host reboot
  is not available to us.
- **NVIDIA GPU Operator:** with the driver, the container toolkit, standalone DCGM and the DCGM exporter **disabled**.
  The host runs `nvidia-dcgm` and its own DCGM exporter. The operator deploys the device plugin and GPU feature
  discovery.
- **NVSentinel:** Janitor and janitor-provider only.
  - Reboots used the `kind` simulator, so no physical reboot was involved anywhere in this report.
  - `GPUReset` ran the Janitor's real reset Job with `hostDriverRootPath: "/"` and `driverRoot: "/"`.
- **The reset:** one `GPUReset` naming one GPU by UUID, on a cordoned and drained tray.

## A. Restore waits for operands that were never deployed

GPUReset conditions (times UTC):

| Type | Status | Reason | Time |
| --- | --- | --- | --- |
| Ready | True | `ReadyForReset` | 13:21:59 |
| ServicesTornDown | True | `ServiceTeardownSucceeded` | 13:22:01 |
| ResetJobCreated | True | `ResetJobCreationSucceeded` | 13:22:01 |
| ResetJobCompleted | True | `ResetJobFailed` | 13:22:52 |
| ServicesRestored | False | `RestoringServices` ("Re-deploying gpu-operator managed services") | 13:22:52 |
| Complete | True | `RestoreTimeoutExceeded` ("failed to restore gpu-operator managed services within the timeout period") | 13:32:53 (`phase: Failed`) |

Janitor log (JSON, fields trimmed):

```text
13:22:52Z INFO  "Managed services re-enabled"                     controller=gpureset
13:32:53Z ERROR "Managed service restoration timeout exceeded"    controller=gpureset
13:32:53Z ERROR "terminal failure"                                controller=gpureset
13:32:53Z INFO  "Node lock successfully released for maintenance resource"
```

In v1.25.0, the `gpu-operator` registry entry lists four apps:
- `nvidia-device-plugin-daemonset`;
- `nvidia-dcgm`;
- `nvidia-dcgm-exporter`;
- `gpu-feature-discovery`.

The registry's `RestoreTimeout` is 10 minutes. On restore, `checkPodsReady` waits until **every** listed app has a
Ready pod on the node, and treats "no pod for this app" as "not ready yet". On this site the operator never deploys
`nvidia-dcgm` or `nvidia-dcgm-exporter`, so their pods can never appear. The restore therefore always runs into its
10-minute timeout: 13:22:52 → 13:32:53 here.

The tray's labels and operator pods were in fact restored. The device plugin and GPU feature discovery came back,
the GPUs stayed healthy (ECC 0) and the tray returned to service. The Janitor still ended `RestoreTimeoutExceeded`
and held the node lock for the full ten minutes.

**Expected:** restore waits only for the apps that had pods on the node when teardown began, and the result
reflects whether those came back.

**Suggested fix:**
1. At teardown, record in status which managed apps had pods on the node. On restore, wait only for those.
2. Document that `serviceManager.spec.apps` (already accepted by the v1.25.0 config loader) can narrow the list to the
   operands a site deploys, and expose it in the chart's values. We have not tried this override.

## B. No pre-flight for a platform whose host cannot reset a single GPU

Separately, on 2026-09-17, we tested GB300 NVL72 trays directly. After a clean teardown of the GPU stack's
per-node components, `nvidia-smi -r -i <gpu>` and a tray-wide `nvidia-smi -r` both answered `Not Supported` for every
GPU. Our platform therefore never asks for a GPU reset on this model and escalates to a reboot instead.

The Janitor has no equivalent check. It accepts the `GPUReset`, tears down the operator's operands, runs the reset
Job, and then pays the restore cost in part A. That means about 11 minutes of disruption for a reset the platform
cannot do.

**Expected:** before teardown, the Janitor checks whether the target GPUs can be reset from the host, and if not ends
at once with a distinct reason, for example `ResetNotSupported`, touching nothing. Two ways to do that:
- a per-GPU-model capability table;
- a dry probe in the reset Job before teardown.

**Suggested fix:**
1. Add a pre-flight step, before `ServicesTornDown`, that runs `nvidia-smi --query-gpu=...` (or the NVML equivalent)
   to find out whether a reset is supported for the targets.
2. Or keep a small per-model table (GB200 / GB300 NVL72: host single-GPU reset not supported) and refuse with
   `ResetNotSupported`.
3. Let fault-remediation map `COMPONENT_RESET` to a reboot on those models.

## Evidence limits

- **The Janitor run does not show the platform limit in part B.** It ran with `driverRoot: "/"`, the same setting
  that on a later HGX B300 run made the Job fail before it reached the host's `nvidia-smi` (`chroot: failed to run
  command 'nvidia-smi': No such file or directory`). We did not capture this Job's log; the capture step timed out.
  The GB300 `ResetJobFailed` above is therefore most likely our misconfiguration, not the platform answering
  `Not Supported`.
- **The `Not Supported` result for part B comes from our separate 2026-09-17 test.** It is recorded in our design
  notes; the raw command output is not in our evidence set.
- **The missing apps in part A are inferred.** The Janitor log lines that would name the app being waited for were
  truncated in our capture. We take the missing apps (`nvidia-dcgm`, `nvidia-dcgm-exporter`) from the site's GPU
  Operator values and the 10-minute timing, which matches the registry's `RestoreTimeout`.

## Impact

- **Part A:** every `GPUReset` on a site that disables any of the four registry operands ends
  `RestoreTimeoutExceeded`, even when the reset itself succeeds, and holds the node lock for ten minutes. In
  v1.25.0, a successful reset on such a site would still be reported as a failure.
- **Part B:** on GB300 NVL72, every `GPUReset` is roughly 11 minutes of operator teardown and restore for a reset the
  platform cannot do.

## Workaround we use

- Our platform never hands a GPU reset to the Janitor on GB300 NVL72. A model-level pin marks host GPU reset as
  unsupported, so a reset recommendation becomes a node reboot.
- It reads any reason other than `GPUResetSucceeded` as a failure (issue 03).
- We have not narrowed `serviceManager.spec.apps` on this site.
