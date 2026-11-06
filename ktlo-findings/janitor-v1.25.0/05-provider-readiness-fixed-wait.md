# janitor: RebootNode `NodeReady=True` does not prove the node rebooted; several providers' `IsNodeReady` is a fixed 5-minute wait

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug / enhancement, janitor, janitor-provider |

## Status on main

**Still present on main@229530ed.** Three cloud providers under `janitor-provider/pkg/csp/` still implement
`IsNodeReady` as a fixed five-minute wait from the signal time, and `janitor/pkg/controller/rebootnode_controller.go`
has no change since v1.25.0 (no boot-ID comparison in the controller). Only the `generic` provider compares boot
IDs.

## Steps to reproduce

(Code-level; we have not observed a false success.)
1. Use one of the providers whose `IsNodeReady` is a fixed wait.
2. Create a `RebootNode` for a node, and make the out-of-band reset request be accepted but not take effect (for
   example, against a test double of the provider API, or by pointing the request at a different instance).
3. Observe after ~5 minutes: the Node is still `Ready` with an unchanged `status.nodeInfo.bootID`, and the
   `RebootNode` ends `NodeReady=True, reason=Succeeded`.

**Component:**
- janitor-provider cloud provider clients (`janitor-provider/pkg/csp/<provider>/` → `IsNodeReady`);
- Janitor RebootNode controller (`handleRebootInProgress` in `janitor/pkg/controller/rebootnode_controller.go`).

## Environment

- **Hardware:** NVIDIA HGX B300, 8 × B300 SXM6 per x86_64 node. Driver 595.91.07.
- **Kubernetes:** a managed Kubernetes service.
- **NVSentinel:** Janitor and janitor-provider only.
- **Provider:** a cloud provider's out-of-band instance reset API.

## Actual behaviour (what happened)

**In our run the result was correct.** We ran a `RebootNode` for a node with a wedged GPU, and it did reboot:

| Time (UTC) | Event |
| --- | --- |
| 01:49:25 | `RebootNode` created |
| 01:49:26 | `SignalSent=True`, reason `Succeeded` (message is the signal timestamp) |
| 01:50:21 | Node `NotReady` (the reset took) |
| 01:55:26 | `NodeReady=True`, reason `Succeeded`, "Node reached ready state post-reboot" — 360 s after the signal |
| 01:55:26–01:55:29 | Node `Ready` with a **new boot ID** (`<boot-id-before>` → `<boot-id-after>`). Our watcher, which polls, recorded 01:55:29; the Janitor's informer already had the node Ready at 01:55:26. |

**What the v1.25.0 source does:**

- This provider's `SendRebootSignal` returns the current time as the request reference. Its `IsNodeReady` parses
  that time and returns `true` once five minutes have passed, without consulting the instance or the node. The code
  comment explains why: the instance does not report a non-running state at once, and Kubernetes still sees the node
  as Ready.
- At least two other cloud providers in `janitor-provider/pkg/csp/` use the same five-minute pattern.
- The Janitor controller combines the provider's answer with the Node's `Ready` condition
  (`cspReady && kubernetesReady`) and re-checks every 60 s.

So for these providers, `NodeReady=True/Succeeded` means "at least five minutes have passed since the signal, and
the Node object is Ready". It does not mean "the node rebooted".

**What can go wrong:** if the provider accepts the reset request but the instance never actually resets, the Node
is still Ready five minutes later, and the `RebootNode` would end `NodeReady=True/Succeeded`. That can happen if the
request is accepted and then dropped, or if the action is applied to the wrong instance.

We have **not** observed such a false success. This finding comes from reading the code, checked against the one
real run above.

By contrast, the `generic` provider stores the node's pre-reboot boot ID as the request reference and reports
ready only once the boot ID has changed and the node is Ready.

## Expected behaviour

`NodeReady=True` with reason `Succeeded` should mean that the node went through a reboot and came back Ready, the
same guarantee the `generic` provider gives. It should not depend on how long the provider waits.

## Impact

- A consumer that treats `NodeReady=True/Succeeded` as "rebooted" can return a node to service that was never
  rebooted, with the fault that triggered the reboot still present.
- This matters most for faults that only a reboot clears, such as a wedged GPU.

## Suggested fix / design direction

1. **Make the check provider-independent in the controller.** Record `node.Status.NodeInfo.BootID` in the
   `RebootNode` status when the signal is sent (for example `status.preRebootBootID`). Require a different boot ID
   and `Ready=True` before setting `NodeReady=True/Succeeded`. The generic provider already does this.
2. **Keep the provider's wait, but only as a floor.** It can stay as a minimum delay before the first check, but the
   provider's answer should come from the instance (state transitions or a restart counter, where the platform
   exposes one), not from the clock alone.
3. **Report a no-reboot outcome.** If the boot ID has not changed by the reboot timeout, end
   `NodeReady=False, reason=NotRebooted` (or `Timeout`) instead of succeeding.

## Workaround we use

Our health platform does not take `NodeReady=True` as proof of repair:

1. It records the node's boot ID before the handoff.
2. It returns the node to service only after the boot ID changes.
3. It then re-runs its passive checks and an acceptance run.
