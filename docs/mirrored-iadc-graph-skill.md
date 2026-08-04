# The mirrored `iadc-graph` skill — source sha and how to refresh it

`iadc-graph/skills/iadc-graph/` in this repo is a **copy** of the canonical skill in
[IADC-Core](https://github.com/teamignyte/IADC-Core) (`.claude/skills/iadc-graph/`), taken at the
sha that built the **deployed** graph image. It carries **zero local patches**. The enclosing
`iadc-graph/` plugin directory additionally holds `.claude-plugin/plugin.json`, which makes it
installable.

| | |
|---|---|
| **Upstream** | `teamignyte/IADC-Core`, path `.claude/skills/iadc-graph/` — *ours*, not a third party |
| **Mirrored at** | `5ef7e16` (IADC-Core `iv-356-per-project-config-parity`, not yet on `main`) — taken 2026-08-04 |
| **Contents** | `iadc-graph/skills/iadc-graph/` holds 7 `.md` files: `SKILL.md` + 6 under `references/` |
| **Local patches** | **none in `iadc-graph/skills/iadc-graph/`.** Any difference from upstream there is staleness — take upstream |

`5ef7e16` is IV-375's fix for this skill's stale per-project config filenames — a prose-only
change, no `graph_mcp/` edit. It was **not** re-verified against the running container the way
`6dc3999` was; instead, against IADC-Core's own deploy log. `.scratch/arch-round3-split/results/
27-deploy-5195a74.log` records a successful graph-image deploy (`DEPLOY_EXIT=0`, `graph /health`
responding) at `5195a74`, dated 2026-08-03 — after `6dc3999`'s 2026-07-31 verification and an
ancestor of `5ef7e16`. `git log 5195a74..5ef7e16 -- graph_mcp/ .claude/skills/iadc-graph/` in
IADC-Core returns only IV-375's own commit, so this refresh — the four files' prose refinement
`6dc3999`'s mirroring had already found and left lagging as the harmless direction, plus IV-375's
two-line fix on top — documents nothing beyond what `5195a74` actually built and deployed.

**The residual is unchanged going forward:** a graph image deployed after `5195a74` would leave
this mirror lagging again, which the ordering rule below permits. Re-verify by content (the
`6dc3999` method) or by deploy log (this refresh's method) before a release that matters.

---

## The ordering rule — it is release-blocking

**The mirrored skill may lag the deployed server, never lead it.** A server tool the skill does not
mention is harmless. A skill promising a tool the deployed server lacks makes Claude call it and
fail.

So: **deploy the graph image first, then refresh this mirror from the sha that built it, then
publish.** Never refresh from IADC-Core `HEAD` — `HEAD` can be ahead of what is deployed, which is
the harmful direction. At the time of writing, `HEAD` *is* ahead: four of the seven files differ.

Refresh is triggered by a graph **deploy**, not by a release schedule.

---

## Refreshing

Mechanical. Nothing in `iadc-graph/skills/iadc-graph/` may be hand-edited — a fix belongs upstream
in IADC-Core, where a drift-guard test couples the skill to the server's real tool roster on every
commit.

```bash
# from IADC-Core, with <sha> = the sha that built the newly deployed graph image
git archive <sha> .claude/skills/iadc-graph \
  | tar -x -C ../IADC-Marketplace/iadc-graph/skills --strip-components=2
```

Then verify the copy is clean and update the table above:

```bash
# byte-identity — the compared subtree holds only mirrored files, nothing to exclude
diff -r iadc-graph/skills/iadc-graph <path-to-IADC-Core-worktree-at-sha>/.claude/skills/iadc-graph
```

The command must print nothing.

## Why a copy at all

A `git-subdir` source pointing straight at IADC-Core would need no copy and would carry a true
`sha` pin — but every installer would then need git read access to IADC-Core, i.e. to the
review-tool source. See the family's
[ADR 0003](https://github.com/teamignyte/IADC/blob/main/docs/adr/0003-shared-skills-ship-as-pinned-marketplace-plugins.md).

Because the mirror is a **relative-path** plugin source rather than a git source, there is no `sha`
field in its marketplace entry to pin. The pin *is* this file: the copy plus the sha recorded above.
