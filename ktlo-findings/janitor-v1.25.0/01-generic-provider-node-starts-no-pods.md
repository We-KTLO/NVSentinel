# janitor-provider: `generic` provider cannot reboot a node that no longer starts pods, and waits the full reboot timeout

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug, janitor-provider |

## Status on main

**Still present on main@229530ed.** Since v1.25.0 the `generic` provider gained a boot-ID guard in the reboot
command (91b76551, #2110) and removal of stale reboot Jobs (ad517238, #2178). `checkRebootJobPodStatus` on main
still skips container states it considers transient (`ContainerCreating`, `PodInitializing`) and pods with no
container status, with no time bound, so an on-node reboot pod that never starts is still reported only at the
RebootNode timeout. The chart's `csp.provider` comment still describes `generic` only as "reboots via privileged
Job", without the precondition.

## Steps to reproduce

1. Deploy the Janitor and janitor-provider with `csp.provider: generic`.
2. Put a node into a state where the API server still sees it `Ready` but the kubelet/runtime starts no new
   container (in our case, a wedged GPU; any pod pinned to the node stays `Pending` with no container status).
3. Create a `RebootNode` for that node.
4. Observe: `SignalSent=True`; the reboot Job's pod never starts; the `RebootNode` stays in progress until the
   reboot timeout (25m default) and ends `NodeReady=False, reason=Timeout`. The node is never rebooted.

Note: step 3-4 is the behaviour expected from the source; see "Actual behaviour" for what we observed directly.

**Component:** janitor-provider, `generic` provider (`janitor-provider/pkg/csp/generic/generic.go`)

## Environment

- **Hardware:** NVIDIA HGX B300, 8 × B300 SXM6 per x86_64 node. Driver 595.91.07, CUDA 13.2.
- **Kubernetes:** a managed Kubernetes service. The NVIDIA GPU Operator runs in device-plugin mode, and the node
  image also runs host-level NVIDIA services.
- **NVSentinel:** Janitor and janitor-provider only. Every monitor, quarantine, drain and remediation module is
  disabled. An external health platform detects faults, cordons and drains the node, then creates the
  `RebootNode`.
- **Provider:** first considered `generic`. The reboot was actually done with a cloud provider's out-of-band reset
  API (see "Workaround we use").

## Actual behaviour (what happened)

One GPU on a node wedged. The site's own detector reported:

```text
HealthcheckUnknown:: GPU count check failed: Command '['nvidia-smi', '--list-gpus']' returned non-zero exit status 255.
HealthcheckUnknown:: NVLink speed check failed: GPU 3 MIG status check exited with code 6; GPU 3 NVLink speed check exited with code 6
Healthcheck:: dcgmi health check failed; Recommended Action: Reboot
```

After that the node started no new pod:

- The health platform's agent DaemonSet pod stayed `Pending` for more than an hour, with no container started.
- A plain pod pinned to the node and requesting no GPU also stayed `Pending`, with no container and no events. We
  saw this by hand; we did not keep that pod object.

The node stayed `Ready` from the API server's point of view. We did not find out why the runtime started no
containers.

The `generic` provider reboots a node by creating a privileged Job pinned to it with `spec.nodeName`
(`SendRebootSignal` → `buildRebootJob`, command `chroot /host reboot`). On this node that pod can never start, so
the reboot can never happen. **This is the node that most needs a reboot.**

We did **not** run a `RebootNode` with the `generic` provider against this node, because we could see it would not
start the pod. What follows comes from reading the v1.25.0 source:

- `IsNodeReady` → `checkRebootJobPodStatus` treats `ContainerCreating` and `PodInitializing` as transient, and skips a
  pod with no container status.
- A reboot pod stuck like the pods above is therefore never reported as failed.
- We expect the `RebootNode` to stay at `SignalSent=True` until the reboot timeout (`25m` by default), then end
  `NodeReady=False, reason=Timeout`, with the node never rebooted.

## Expected behaviour

- A `RebootNode` on such a node either reboots it, or fails fast with a reason that says the on-node reboot pod
  could not start.
- The documentation for `csp.provider: generic` says that it needs a node that can still start a privileged pod.

## Impact

- On a fleet using `generic`, the fault class that most needs a reboot cannot be repaired by the Janitor: a wedged
  GPU that stops the container runtime from starting pods.
- The `RebootNode` waits for the full reboot timeout before it reports anything. A consumer that reads only
  `SignalSent=True` may believe a reboot is under way.

## Suggested fix / design direction

1. **Fail fast:** in `checkRebootJobPodStatus`, treat a reboot pod that is still unstarted after a bounded time (no
   container status, or `ContainerCreating`, for longer than N seconds) as a failure, with a distinct reason such as
   `RebootPodNotStarted`. The `RebootNode` can then end `NodeReady=False` well before the 25-minute timeout.
2. **Fallback path:** let a site configure a second, out-of-band provider, used when the on-node Job does not start
   within N seconds. For bare metal this could be a BMC power action over Redfish or IPMI; for a cloud, that cloud's
   API.
3. **Docs:** state the precondition in the chart's `csp.provider` comment and in the provider docs.

## Workaround we use

On fleets where this fault is possible we use a provider that reboots out of band through the platform's API, not
`generic`, and keep `generic` for sites that have nothing else. On this node the out-of-band reset worked:

- `SignalSent=True/Succeeded` at 01:49:26 UTC.
- The node went `NotReady` at 01:50:21 and was `Ready` again at about 01:55:26, with a new boot ID.
- `NodeReady=True/Succeeded` at 01:55:26.
- After the boot the GPUs and the site's checks were healthy.
