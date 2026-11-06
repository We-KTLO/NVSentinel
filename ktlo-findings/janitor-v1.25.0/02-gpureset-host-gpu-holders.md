# janitor: GPUReset does not quiesce host-level GPU holders (host `nv-hostengine`, `nvidia-persistenced`); `nvidia-smi --gpu-reset` fails with "In use by another client"

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug / enhancement, janitor, gpu-reset |

## Status on main

**Partly addressed on main@229530ed; the host-level case is still present.** 598ce594 (#2276, "stop GPU client
pods outside the GPU Operator before a GPU reset") makes the controller stop *pods* that hold the GPUs, via the
`nvidia.com/gpu.deploy.client` node label (GPU Operator 26.7+). Its documentation (`docs/configuration/janitor.md`,
"GPU clients") states explicitly: "The controller does not stop processes that run on the host outside
Kubernetes, for example `nvidia-persistenced` or a host DCGM." `gpu-reset/gpu_reset.sh` on main still has no
step that stops host services and no holder pre-flight, and the `ResetJobCompleted` message is still the fixed
string "GPU reset failed". Re-scope the issue to host (systemd) holders and reference #2276 when filing.

## Steps to reproduce

1. Use a node image that runs `nvidia-dcgm.service` (host `nv-hostengine`) and/or `nvidia-persistenced` under
   systemd, with the GPU Operator in device-plugin mode and a host-installed driver.
2. Deploy the Janitor with `gpuReset.resetJob.hostDriverRootPath: "/"` and `driverRoot: "/host"`.
3. Cordon and drain the node, then create a `GPUReset` naming one GPU by UUID.
4. Observe: operator operands are torn down, the reset Job runs, `nvidia-smi --gpu-reset` refuses with
   "In use by another client"; the GPUReset ends `ResetJobFailed` with message "GPU reset failed".

**Component:**
- Janitor GPUReset controller (`janitor/pkg/controller/gpureset_controller.go`);
- the `gpu-operator` service manager (`janitor/pkg/gpuservices/manager.go`);
- the reset script (`gpu-reset/gpu_reset.sh`).

## Environment

- **Hardware:** NVIDIA HGX B300, 8 × B300 SXM6 per x86_64 node. Driver 595.91.07, CUDA 13.2. NVSwitch baseboard with
  Fabric Manager on the host.
- **Kubernetes:** a managed Kubernetes service. The NVIDIA GPU Operator runs in device-plugin mode and deploys the
  device plugin, standalone DCGM, the DCGM exporter and GPU feature discovery.
- **Node image:** also runs host-level NVIDIA services under systemd:
  - `nvidia-dcgm.service` (an `nv-hostengine`), which the site's own health checks use;
  - `nvidia-persistenced`;
  - Fabric Manager.
- **NVSentinel:** Janitor and janitor-provider only. `gpuReset.resetJob.hostDriverRootPath: "/"` and
  `driverRoot: "/host"` (host-installed driver).
- **The reset:** a `GPUReset` naming one GPU by UUID (`spec.selector.uuids: [GPU-<uuid>]`), on a cordoned and drained
  node.

## Actual behaviour (what happened)

The Janitor tore down the GPU Operator's operands and ran the reset Job. The host's `nvidia-smi` then refused the
reset because the device was in use.

Reset Job log (identifiers redacted):

```text
(2026-10-13 02:04:20) INFO: Using DRIVER_ROOT=/host
(2026-10-13 02:04:20) INFO: Testing nvidia-smi invocation: chroot /host nvidia-smi --version
NVIDIA-SMI version  : 595.91.07
(2026-10-13 02:04:21) INFO: Targets:
  GPU-<uuid>
(2026-10-13 02:04:21) INFO: Resetting GPUs...
(2026-10-13 02:04:22) ERROR: Reset failed. See details below:
The following GPUs could not be reset:
  GPU <pci-bus-id>: In use by another client
1 device is currently being used by one or more other processes (e.g., Fabric Manager, CUDA application, graphics application such as an X server, or a monitoring application such as another instance of nvidia-smi). Please first kill all processes using this device and all compute applications running in the system
```

GPUReset conditions:

| Type | Status | Reason | Time (UTC) |
| --- | --- | --- | --- |
| Ready | True | `ReadyForReset` | 02:04:16 |
| ServicesTornDown | True | `ServiceTeardownSucceeded` ("gpu-operator managed services have been removed") | 02:04:18 |
| ResetJobCreated | True | `ResetJobCreationSucceeded` | 02:04:18 |
| ResetJobCompleted | True | `ResetJobFailed` ("GPU reset failed") | 02:09:18 |
| ServicesRestored | True | `ServiceRestoreSucceeded` | 02:09:20 |
| Complete | True | `ResetJobFailed` | 02:09:20 (`phase: Failed`) |

### The processes holding the device

We listed the processes holding the GPU's device node on the same node, by hand (cgroup paths shortened, pod UIDs
removed):

```text
== processes holding /dev/nvidia3, /dev/nvidiactl, /dev/nvidia-uvm
dcgm-exporter   [/dev/nvidia-uvm /dev/nvidia3 /dev/nvidiactl]  cgroup=/kubepods.slice/.../<container>
nvidia-device-p [/dev/nvidia-uvm /dev/nvidia3 /dev/nvidiactl]  cgroup=/kubepods.slice/.../<container>
nv-hostengine   [/dev/nvidia-uvm /dev/nvidia3 /dev/nvidiactl]  cgroup=/kubepods.slice/.../<container>
nv-hostengine   [/dev/nvidia-uvm /dev/nvidia3 /dev/nvidiactl]  cgroup=/system.slice/nvidia-dcgm.service
nvidia-persiste [/dev/nvidia3 /dev/nvidiactl]                  cgroup=/system.slice/nvidia-persistenced.service
```

This snapshot was **not** taken while the reset ran: the GPU Operator's pods are in it, so it was taken while they
were running. While the reset ran, those pods were gone:
- `ServicesTornDown=True`;
- the controller's drift check (`getOrCreateJob` → `checkPodsTerminated`) creates the Job only after the operator
  pods have terminated.

So the device was most likely held by the two host-level processes: the host `nv-hostengine` and
`nvidia-persistenced`. We did not establish which of the two (or both) blocked the reset. Fabric Manager runs on
this host but was not holding the device.

### What the v1.25.0 source does

- The `gpu-operator` service manager tears down and restores four operands: device plugin, `nvidia-dcgm`,
  `nvidia-dcgm-exporter`, GPU feature discovery. It does this through `nvidia.com/gpu.deploy.*` node labels.
- `gpu_reset.sh` runs `nvidia-smi --gpu-reset -i <targets>` with no step that stops a host service, and the Job has
  no pre-reset hook.

We did not stop the site's host services to retry; that was outside what we had agreed to do on the node.

## Expected behaviour

Either of:

- A way to quiesce host-level GPU holders for the duration of the reset, and restart them after it.
- A pre-flight that sees the device is still held after teardown and fails with a clear reason (for example
  `DeviceInUse`, naming the holders), instead of a generic `ResetJobFailed` with "GPU reset failed".

## Impact

- On any node image that runs DCGM or `nvidia-persistenced` on the host (common when the host engine serves more
  than Kubernetes), `GPUReset` cannot succeed.
- Each failed attempt still tears down and restores the GPU Operator's operands. The DCGM exporter disappears from
  monitoring for that time, and the node keeps its fault.

## Suggested fix / design direction

1. **Host-services hook in the reset Job.** Add an optional list of host systemd units to stop before
   `--gpu-reset` and start again afterwards, run with `chroot /host systemctl`, which the Job's privileges already
   allow. For example `gpuReset.resetJob.hostServices: [nvidia-dcgm, nvidia-persistenced]`. Restart the units on
   every exit path.
2. **Pre-flight check.** After teardown, list the processes that hold the target devices (`nvidia-smi
   --query-compute-apps`, plus a scan of `/proc/*/fd` for `/dev/nvidia<N>`). If any remain, end with a distinct reason
   whose message names the holders.
3. **Surface the cause.** Copy the first error line of the reset output into the `ResetJobCompleted` /
   `Complete` condition message. Related: issue 03 of this set.

## Workaround we use

None that resets the GPU. Our health platform:
- refuses a reset handoff unless the faulted GPUs are named (an empty selector resets every GPU);
- reads this outcome as a failure (see issue 03);
- escalates to a node reboot through an out-of-band provider.
