# finops-sre-agent-pack

[![Install to Azure SRE Agent](https://img.shields.io/badge/Install-Azure%20SRE%20Agent-0078D4?logo=microsoftazure&logoColor=white)](https://tomkerkhove.github.io/azure-sre-agent-plugin-installer/?repo=nirmash%2Ffinops-sre-agent-pack)

FinOps capabilities for [Azure SRE Agent](https://azure.microsoft.com/en-us/products/sre-agent),
packaged as **skills + agents** — not as changes to the SRE Agent product. The pack and its agent use
read-only Azure tools. Budget planning can produce a governed shell script for a human to review,
save, and run manually with their own permissions; the agent never executes it.

Structure mirrors [Azure/sre-agent-plugins](https://github.com/Azure/sre-agent-plugins).

> **Note:** These skills are designed for use with the Azure SRE Agent and may not work with other
> coding agents.

## Included plugin

| Plugin | Description |
|--------|-------------|
| [`finops`](plugins/finops/README.md) | **FinOps pack** — ten read-only FinOps skills including `finops-managed-scope` and deterministic `finops-report-renderer`, the standalone `finops-investigator` agent, and seven proactive read-only scheduled tasks including six Live Reports. Budget planning can generate a validated human-run application script. |

## Managed scope

Every FinOps run dynamically reads
`properties.knowledgeGraphConfiguration.managedResources` from the Azure SRE Agent ARM resource
identified by `AGENT_RESOURCE_ID`. Subscription, resource-group, and management-group scopes are
supported. Change the agent's managed resources and the new boundary takes effect on the next run;
the plugin does not need to be reinstalled.

- Scheduled tasks are hard-bound to the current managed scope and fail closed if discovery or
  management-group expansion fails. They do not query analysis data, send email, or save a report
  after a scope-discovery failure.
- Interactive requests use managed scope by default. A request outside it proceeds only after the
  exact outside scopes are shown and the user explicitly confirms them on a subsequent turn.
- UsageDetails-based reports disclose included, excluded, and unattributed row counts and costs so
  scope coverage is visible.
- Broad inherited or historical RBAC does not broaden this logical boundary. Installation does not
  revoke old broad grants.

This is a skill-and-policy addition only; it makes no SRE Agent runtime changes.

## Install

The supported installer installs the complete **`finops` plugin package** from this repository: all
ten skills, `finops-investigator`, seven scheduled tasks, and six Live Reports. It does not install
unrelated plugins from the marketplace.

The installer uses the SRE Agent management API and requires `az`, `curl`, and `python3`.
`AGENT_RESOURCE_ID` is required; endpoint-only installation is not supported because the live
managed scope must be discoverable. Sign in with `az login` using an identity that can manage the
agent and create role assignments on it, then run:

```bash
AGENT_RESOURCE_ID=/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.App/agents/<agent> \
  ./plugins/finops/install-api.sh
```

While this repository is private, provide a GitHub token so the agent service can clone it:

```bash
GITHUB_PAT="$(gh auth token)" \
AGENT_RESOURCE_ID=/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.App/agents/<agent> \
  ./plugins/finops/install-api.sh
```

The installer grants the agent's user-assigned identity **Reader** on the exact agent ARM resource
so each run can read `managedResources`. To optionally grant **Cost Management Reader** on the
minimum scopes required by the UsageDetails transport, also set
`MI_OBJECT_ID=<agent-mi-object-id>`. For an RG-managed boundary, Azure exposes UsageDetails only at
the containing subscription endpoint, so the role is granted at that subscription while the pack
still filters and reports only the exact managed RGs. It does not revoke older broad assignments or
add budget-write permissions.
`SUB_ID` is deprecated and ignored for scheduled scope. See the
[FinOps plugin documentation](plugins/finops/README.md#install-everything-recommended) for all
configuration options and installed components.

## Contributing another plugin

This section is for repository contributors, not an additional installation step:

1. Add the new plugin directory under `plugins/`.
2. Add its marketplace entry to [`.github/plugin/marketplace.json`](.github/plugin/marketplace.json).
3. Provide a plugin-specific installer or extend the package installer intentionally; the existing
   `plugins/finops/install-api.sh` installs only the `finops` package.

## Design & requirements background

The feature catalog, live data-source validation, and customer-requirement mapping that motivate
this pack live in the product-research doc (`finops-sre-agent-product-research.md`, §5A and §10).
All 13 FinOps data sources were validated live against a deployed SRE Agent before any skill was
written.

## Test

```bash
pip install -r requirements-dev.txt
pytest tests/          # 281 offline tests, including managed-scope and budget-script coverage
```
