# IADC-Marketplace

The **`ignyte`** Claude Code plugin catalog — how the [IADC](https://github.com/teamignyte/IADC)
family is distributed.

```bash
claude plugin marketplace add https://github.com/teamignyte/IADC-Marketplace.git --scope project
claude plugin install iadc@ignyte --scope project
```

That installs the whole suite. Install a single product instead if that's all you need:

| Plugin | What you get |
|---|---|
| **`iadc`** | Everything — no skills of its own; it exists only to pull in the two below |
| **`iadc-advisor`** | Advisory Appian architect: pressure-test tickets, produce build specs. Plans; never builds |
| **`iadc-tester`** | Sync an Appian app's Selenium test suite with a Jira ticket's requirements |
| **`iadc-graph`** | Query the App Graph. **Installed automatically** — you never install it directly |

`iadc-advisor` and `iadc-tester` both declare `iadc-graph` as a dependency, so it arrives with
whichever you install, and disabling it while either is enabled is refused.

## What lives here

```
.claude-plugin/marketplace.json    the catalog
iadc/                              the bundle — a manifest and nothing else
iadc-graph/                        the mirrored graph skill (SKILL.md + references/)
docs/mirrored-iadc-graph-skill.md  where the mirror came from, and how to refresh it
```

Only two plugins are *stored* here. `iadc-advisor` and `iadc-tester` are fetched from their own
repositories — the catalog just points at them.

> **`iadc-advisor` is pinned to `ref: iadc-advisor-plugin`, and that is a stopgap.** Its repo's
> default branch, `main`, is the *pre-plugin* layout — it has `deliverable/` and no
> `iadc-advisor/` directory — so an unpinned entry fetches nothing and the plugin fails to
> resolve. The real fix is to merge `iadc-advisor-plugin` into `main`, which is 25 commits of the
> entire plugin architecture that were never merged. **Drop the `ref` once that lands.**

**This repo is client-facing.** Adding a marketplace clones the whole repository, so everyone who
installs anything can read everything here. Family decisions and internal tooling deliberately live
in the [umbrella](https://github.com/teamignyte/IADC) instead, which nobody clones
([ADR 0001](https://github.com/teamignyte/IADC/blob/main/docs/adr/0001-iadc-family-is-five-repos-in-two-tiers.md)).

## The graph mirror

`iadc-graph/` is a copy of IADC-Core's canonical skill, taken at the sha that built the **deployed**
graph image — never at `HEAD`, which can be ahead of what is running. A skill that promises a tool
the deployed server lacks makes Claude call it and fail.

Never hand-edit it. Fix it upstream in IADC-Core, where a drift-guard test binds it to the server's
real tool roster per commit, deploy, then refresh. Procedure and current sha:
[`docs/mirrored-iadc-graph-skill.md`](docs/mirrored-iadc-graph-skill.md).

## Adding a plugin

Add its entry to `.claude-plugin/marketplace.json`. **Never declare `skills` or `hooks` in a
plugin's `plugin.json`** — both are auto-discovered, and declaring them as well registers the same
paths twice, after which the plugin installs successfully but loads nothing. `claude plugin
validate` passes on that broken manifest, so it is not a gate; the real check is a live install
reporting the plugin as `enabled` in `claude plugin list`.
