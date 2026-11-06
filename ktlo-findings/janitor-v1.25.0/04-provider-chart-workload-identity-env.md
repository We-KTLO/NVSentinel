# janitor-provider chart: workload-identity path for `<provider>` is missing two SDK environment variables

> Draft GitHub issue for NVIDIA/NVSentinel. Not filed. Copy the H1 as the issue title.

| | |
| --- | --- |
| **Affected version** | v1.25.0 (chart and images `v1.25.0`, commit `0485d38`) |
| **Checked against** | `main@229530ed` (2026-11-05) |
| **Suggested labels** | bug, helm, janitor-provider |

## Status on main

**Still present on main@229530ed.** The `<provider>` block of
`distros/kubernetes/nvsentinel/charts/janitor-provider/templates/deployment.yaml` is unchanged since v1.25.0 (it
still sets only region, account scope, credentials file and profile). Changes to this template since v1.25.0 are
`LOG_LEVEL` (4fa51acb, #2074) and a new provider's block (5fa18dd9, #2114).

## Steps to reproduce

1. Install the janitor-provider chart with `csp.provider: <provider>`, region and account scope set, and
   `credentialsFile` empty (workload identity bound to the `janitor-provider` service account).
2. Do not add anything to `extraEnv`.
3. Expected from the source (not captured, see "Evidence limits"): the provider fails to build the
   workload-identity configuration provider and exits with `Failed to create CSP client`.
4. Add `<SDK_PRINCIPAL_VERSION_VAR>` and `<SDK_PRINCIPAL_REGION_VAR>` via `extraEnv`: the provider
   authenticates and `RebootNode` succeeds.

> **Before filing:** this draft names neither the cloud nor its variables. Upstream cannot act on it until the
> filer adds the `csp.provider` value and the two variable names (placeholders below). The filer decides whether
> to name them.

**Component:**
- janitor-provider Helm chart (`distros/kubernetes/nvsentinel/charts/janitor-provider/templates/deployment.yaml`);
- the matching cloud provider client (`janitor-provider/pkg/csp/<provider>/`).

## Environment

- **Hardware:** NVIDIA HGX B300, 8 × B300 SXM6 per x86_64 node. Driver 595.91.07.
- **Kubernetes:** the cloud's managed Kubernetes service.
- **NVSentinel:** Janitor and janitor-provider only.
- **Provider:** `janitor-provider.csp.provider: <provider>`, which reboots through the cloud's out-of-band instance
  reset API. Values:
  - `csp.<provider>.region` and the account-scope value are set;
  - `credentialsFile` is empty, so the client uses the platform's workload identity, bound to the
    `nvsentinel/janitor-provider` service account.

## Actual behaviour (what happened)

With `credentialsFile` empty, the provider client builds its credentials from the SDK's workload-identity
configuration provider. That path reads two environment variables:

- a principal-version variable (`<SDK_PRINCIPAL_VERSION_VAR>`);
- a region variable of its own (`<SDK_PRINCIPAL_REGION_VAR>`). This is a different name from the region variable
  the chart sets.

For this provider, the v1.25.0 chart template sets only:
- the provider's region variable, from `csp.<provider>.region`;
- the account-scope variable;
- the credentials-file variable, when one is given;
- the profile variable, when one is given.

It sets neither of the two variables above. We had to add both through `janitor-provider.extraEnv`:

```yaml
janitor-provider:
  csp:
    provider: <provider>
    <provider>:
      region: <region>
      <account-scope-key>: <redacted>
  extraEnv:
    - {name: <SDK_PRINCIPAL_VERSION_VAR>, value: "<version the SDK expects>"}
    - {name: <SDK_PRINCIPAL_REGION_VAR>,  value: <region>}
```

With these set, the provider authenticated and the reboot worked: `SignalSent=True/Succeeded`, the node came back
with a new boot ID, and `NodeReady=True/Succeeded`.

**Evidence limits:**
- We did not keep a provider log from a start without the two variables, so we cannot quote the SDK error.
- Reading `janitor-provider/main.go` and `pkg/csp/client.go`, a configuration-provider error surfaces at startup as
  `Failed to create CSP client` and the provider exits, so we expect a crash-looping janitor-provider pod. We have
  not reproduced that.
- What we can show is our overlay and the v1.25.0 chart template.

## Expected behaviour

When the chart selects workload identity (no credentials file), it should set every variable that path needs. The
region variable can be derived from `csp.<provider>.region`; the version variable can be a documented default.
Failing that, the provider should fail at startup with a message naming the missing variables.

## Impact

- A site that follows the chart's documented values for workload identity gets a janitor-provider that cannot
  authenticate, and no reboot or terminate request can succeed until someone finds the missing variables in the SDK.
- Because the reboot path is the fallback for nodes that can no longer start pods (issue 01), this tends to surface
  at the moment a reboot is most needed.

## Suggested fix / design direction

1. In the chart's `<provider>` block, when `credentialsFile` is empty, also emit the version variable (overridable
   value with a documented default) and the SDK's own region variable, set from `csp.<provider>.region`.
2. In the provider client, check before constructing the workload-identity configuration provider that the
   variables it reads are present. Return an error that names them.
3. Add a chart render test that renders `csp.provider: <provider>` with no credentials
   file and asserts both variables are present.
4. Document the workload-identity prerequisites, including the policy that grants only the instance power action
   to the Janitor's service account.

## Workaround we use

Pass both variables through `janitor-provider.extraEnv`, as above. Before relying on a provider, we also review
which variables its SDK reads.
