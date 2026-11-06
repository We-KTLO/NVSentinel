# NVSentinel v1.25.0 Janitor: findings from live runs (draft issues, not filed)

Each file in this directory is a draft GitHub issue for NVIDIA/NVSentinel: an H1 title, the affected version, its
status on `main`, steps to reproduce, actual and expected behaviour, and a proposed fix. None has been filed.

They come from live runs that drove NVSentinel's Janitor through a custom-resource handoff from an external
GPU-health platform, on two clusters:

- **GB300 NVL72**, 2026-10-03. Reboots went through the **`kind` simulator**. GPUReset ran the real reset Job.
- **HGX B300**, 2026-10-12 and 2026-10-13. One real out-of-band reboot of a faulted node, and three real
  `GPUReset` attempts.

Identifiers are redacted throughout: hostnames, IPs, rack and tenant names, account scopes, UUIDs, boot IDs and CR
names. The cloud provider is described generically.

**Affected version:** v1.25.0 (commit `0485d38`). **Re-checked against:** `main@229530ed` (2026-11-05), by reading
the code; nothing was re-run on `main`.

| # | File | Status on `main@229530ed` | One line |
| --- | --- | --- | --- |
| 01 | [`01-generic-provider-node-starts-no-pods.md`](01-generic-provider-node-starts-no-pods.md) | Still present | The `generic` provider reboots through a privileged pod on the node, so it cannot reboot a node that has stopped starting pods; a stuck reboot pod is treated as transient until the 25-minute timeout. |
| 02 | [`02-gpureset-host-gpu-holders.md`](02-gpureset-host-gpu-holders.md) | Partly addressed by 598ce594 (#2276) for pod GPU clients; host-level holders still present | GPUReset tears down only GPU Operator pods. A host `nv-hostengine` and `nvidia-persistenced` still held the device, and `nvidia-smi --gpu-reset` refused with "In use by another client". |
| 03 | [`03-gpureset-complete-true-on-failure.md`](03-gpureset-complete-true-on-failure.md) | Still present | A failed GPUReset ends `Complete=True` with a failure reason (`ResetJobFailed`, `RestoreTimeoutExceeded`). Condition-only consumers see success, including NVSentinel's own fault-remediation as documented (code reading). |
| 04 | [`04-provider-chart-workload-identity-env.md`](04-provider-chart-workload-identity-env.md) | Still present | The janitor-provider chart does not set the two variables the provider SDK's workload-identity path reads; we passed them through `extraEnv`. |
| 05 | [`05-provider-readiness-fixed-wait.md`](05-provider-readiness-fixed-wait.md) | Still present | Some cloud providers' `IsNodeReady` is a fixed 5-minute wait. Combined with Node `Ready`, `NodeReady=True` does not prove the node rebooted; nothing checks the boot ID. |
| 06 | [`06-gpureset-unsupported-platform-and-restore-wait.md`](06-gpureset-unsupported-platform-and-restore-wait.md) | Part A appears fixed by 3c637ad4 (#1989); part B still present and **must be re-run before filing** | On GB300 NVL72 there is no pre-flight for a host that cannot reset a single GPU. The restore also waited for GPU Operator operands the site never deployed, so every reset ended `RestoreTimeoutExceeded` after 10 minutes. |

## Evidence strength per finding

- **01: inferred, not observed.**
  - We never ran a `generic` `RebootNode` against the wedged node; we switched providers first. "A `generic`
    reboot could never run there" follows from the provider's design (a Job pinned by `nodeName`) and from the node
    starting no pod. It was not observed.
  - The "plain pod that asks for no GPU stayed Pending" was seen by hand. The pod object was not captured; only
    narrative notes and the agent pod's state record it.
  - The "stuck until the 25-minute timeout" behaviour comes from reading the code.
- **02: the holder snapshot is not from the reset window.**
  - The holder listing also shows the GPU Operator's own pods (`dcgm-exporter`, the device plugin, a pod
    `nv-hostengine`), so it was taken while those were running, not during the reset.
  - That the host `nv-hostengine` and `nvidia-persistenced` were the holders at reset time is an inference: the
    operator pods had been torn down, and the controller's drift check confirms that. We did not isolate which of
    the two blocked the reset.
  - There was a single attempt, on one GPU on one node, that reached `nvidia-smi`.
- **03: solid for the Janitor.**
  - Four objects, all `Complete=True` with `phase: Failed`.
  - The fault-remediation consequence is code reading only; fault-remediation was off.
  - The three B300 resets were created by hand from a template, not through the health platform's handoff.
- **04: no failure was captured.**
  - The evidence is the overlay we deployed (with its comment) and the v1.25.0 chart template; the template does
    confirm that only region, account scope, credentials file and profile are set.
  - There is no provider log from a start without the variables, so the issue cannot quote the SDK error.
  - The issue cannot be acted on upstream until the filer names the provider and the two variables (placeholders
    are in the draft).
- **05: no false success was observed.**
  - The provider's check is a fixed wait, but the Janitor controller does read the node: it ANDs the provider's
    answer with the Node's `Ready` condition. What is missing is a check that the node rebooted (boot ID).
  - In the one real run the node did reboot (new boot ID), and `NodeReady=True` came 360 s after the signal, once
    the node was Ready. The risk is from code reading.
  - The GB300 RebootNode results prove nothing about readiness. The `kind` simulator answers ready at random
    (about 94 % of calls).
- **06: the weakest.**
  - The GB300 reset Job recorded `ResetJobFailed`, but that run used `driverRoot: "/"`, the setting that on B300
    later made the Job fail before reaching the host's `nvidia-smi`. The Job log was not captured: the capture
    holds only `error: timed out waiting for the condition`.
  - The platform limit itself comes from a separate 2026-09-17 test, recorded in our design notes, with no raw
    output in our evidence set.
  - The restore-timeout half (part A) is well supported by the 10-minute timing, the source and the site's GPU
    Operator values (DCGM and DCGM exporter disabled). The captured Janitor log lines were truncated, though, so no
    line names the app being waited for. Part A now appears fixed on `main` (see the file).
  - **Before filing part B:** re-run one GB300 `GPUReset` with `driverRoot: /host` and keep the Job log, or do not
    file part B.

## Seen in the same runs, not drafted as issues

- **Reset image pull counts against the Job deadline.** On the first B300 attempt the reset image pull took 4 min
  29 s of the Job's 300 s deadline, and the Job then failed. A pre-pull, or a deadline that starts at container
  start, would avoid it.
- **Teardown removes the DCGM exporter.** Each GPUReset attempt makes the node's DCGM metrics vanish for the
  duration, which monitoring reads as an exporter outage. This is worth a note in the GPUReset docs.
- **What worked as designed:**
  - The Janitor's `nodes.exclusions` webhook refused a `RebootNode` for an unlabelled node (GB300).
  - The out-of-band reboot of the wedged B300 node worked end to end.
