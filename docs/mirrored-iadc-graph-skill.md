# The mirrored `iadc-graph` skill — source sha and how to refresh it

`iadc-graph/skills/iadc-graph/` in this repo is a **copy** of the canonical skill in
[IADC-Core](https://github.com/teamignyte/IADC-Core) (`.claude/skills/iadc-graph/`), taken at the
sha that built the **deployed** graph image. It carries **zero local patches**. The enclosing
`iadc-graph/` plugin directory additionally holds `.claude-plugin/plugin.json`, which makes it
installable.

| | |
|---|---|
| **Upstream** | `teamignyte/IADC-Core`, path `.claude/skills/iadc-graph/` — *ours*, not a third party |
| **Mirrored at** | `b461811` (IADC-Core `iv-356-per-project-config-parity`, not yet on `main`) — taken 2026-08-04 |
| **Contents** | `iadc-graph/skills/iadc-graph/` holds 7 `.md` files: `SKILL.md` + 6 under `references/` |
| **Local patches** | **none in `iadc-graph/skills/iadc-graph/`.** Any difference from upstream there is staleness — take upstream |

**This row owes a re-record.** `b461811` lives on an unpushed feature branch — unresolvable from
any other clone, and dead if that branch is rebased or squashed when IV-375 merges. Re-record this
row at the post-merge `main` sha once it lands; until then, treat this pin as provisional.

`b461811` is IV-375's fix for this skill's stale per-project config filenames (two commits: the
corrected filenames/command reference, then a follow-up wording fix) — no `graph_mcp/` edit in
either. It was **not** re-verified against the running container the way `6dc3999` was; instead,
against IADC-Core's own deploy log: `.scratch/arch-round3-split/results/27-deploy-5195a74.log`
records a successful graph-image deploy (`DEPLOY_EXIT=0`, `graph /health` responding) at `5195a74`,
dated 2026-08-03 — after `6dc3999`'s 2026-07-31 verification and an ancestor of `b461811`.
`git log 5195a74..b461811 -- graph_mcp/ .claude/skills/iadc-graph/` in IADC-Core returns only
IV-375's two commits, so nothing this refresh carries documents behaviour beyond what `5195a74`
actually built and deployed.

That "nothing beyond" claim covers more than IV-375's own two lines, because a refresh is a
wholesale directory copy, not a cherry-pick, so it also carries forward what `6dc3999`'s mirroring
had already found IADC-Core `HEAD` ahead by. That prior gap was **not** uniformly "prose
refinement" — only `references/identifiers-and-discovery.md` was a pure rename
(`_boundary_node_id` → `ResolvedReference.boundary_node_id()`, no behaviour change). `SKILL.md`,
`references/return-shapes-and-errors.md` and `references/session-lifecycle.md` carried substantive,
client-facing additions instead: a hard constraint on `seed(export_ref=...)` over HTTP with two
allowed roots (IV-321), a second `ValueError`/`ToolError` error class distinct from the
`{"error": ...}` shape, a new guarded `session does not belong to this caller` string, nullable
`kind` (IV-250), and a background TTL sweeper (IV-298) that reverses an earlier "not by a
background sweeper" claim. All of it predates `5195a74` — the same `git log` above covers these
files too — so it is already live on the deployed server, not merely harmless to lag.

**The residual is unchanged going forward:** a graph image deployed after `5195a74` would leave
this mirror lagging again, which the ordering rule below permits. Re-verify by content (the
`6dc3999` method) or by deploy log (this refresh's method) before a release that matters.

---

## The ordering rule — it is release-blocking

**The mirrored skill may lag the deployed server, never lead it.** A server tool the skill does not
mention is harmless. A skill promising a tool the deployed server lacks makes Claude call it and
fail.

So: **deploy the graph image first, then refresh this mirror from the sha that built it, then
publish.** Refreshing straight from IADC-Core `HEAD` is the harmful direction **unless** every
commit between the last deploy sha and `HEAD` is checked, for both `graph_mcp/` and
`.claude/skills/iadc-graph/`, and every one of them is doc-only — never a behaviour change:
`git log <deploy-sha>..HEAD -- graph_mcp/ .claude/skills/iadc-graph/`, read every commit it lists,
don't just count them. IV-375's refresh (`b461811`) is the worked example: the deploy log at
`.scratch/arch-round3-split/results/27-deploy-5195a74.log` fixes the deploy sha at `5195a74`, and
that `git log` lists only the two prose-only commits this ticket made. Skip the check, or find a
behaviour commit in the list, and the default stands: deploy first, refresh from the sha that built
it, publish.

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
