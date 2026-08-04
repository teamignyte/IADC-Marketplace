# The mirrored `iadc-graph` skill

`iadc-graph/skills/iadc-graph/` in this repo is a **copy** of the canonical skill authored in
[IADC-Core](https://github.com/teamignyte/IADC-Core) (`.claude/skills/iadc-graph/`), taken at the
sha that built the **deployed** graph image — or, when the maintainer procedure's check establishes
the deployed server hasn't moved, at `HEAD` directly. It carries **zero local patches** — verified
byte-identical (`diff -r`) against the upstream sha at each refresh, so any difference from
upstream here is staleness, never a deliberate change. The enclosing `iadc-graph/` plugin directory
additionally holds `.claude-plugin/plugin.json`, which makes it installable.

**Never hand-edit anything under `iadc-graph/skills/iadc-graph/`.** A fix belongs upstream in
IADC-Core, where a drift-guard test binds the skill to the graph server's real tool roster on every
commit; from there it is refreshed into this copy and re-verified byte-identical. The refresh
procedure — how the source sha is chosen and re-verified, and the current pin — is a maintainer
concern and lives in IADC-Core's `docs/marketplace-mirror-refresh.md`, not in this client-facing
repo.

## Why a copy at all

A `git-subdir` source pointing straight at IADC-Core would need no copy and would carry a true
`sha` pin — but every installer would then need git read access to IADC-Core, i.e. to the
review-tool source. See the family's
[ADR 0003](https://github.com/teamignyte/IADC/blob/main/docs/adr/0003-shared-skills-ship-as-pinned-marketplace-plugins.md).

Because the mirror is a **relative-path** plugin source rather than a git source, there is no `sha`
field in its marketplace entry to pin. The pin is recorded upstream instead, in IADC-Core's
maintainer procedure.
