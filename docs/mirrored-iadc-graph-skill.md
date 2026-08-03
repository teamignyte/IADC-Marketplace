# The mirrored `iadc-graph` skill — source sha and how to refresh it

`iadc-graph/` in this repo is a **copy** of the canonical skill in
[IADC-Core](https://github.com/teamignyte/IADC-Core) (`.claude/skills/iadc-graph/`), taken at the
sha that built the **deployed** graph image. It carries **zero local patches** — the only additions
are `.claude-plugin/plugin.json`, which makes the directory a plugin, and nothing else.

| | |
|---|---|
| **Upstream** | `teamignyte/IADC-Core`, path `.claude/skills/iadc-graph/` — *ours*, not a third party |
| **Mirrored at** | `6dc3999` (IADC-Core `main`) — taken 2026-08-03 |
| **Contents** | 7 `.md` files: `SKILL.md` + 6 under `references/` |
| **Local patches** | **none.** Any difference is staleness — take upstream |

`6dc3999` was **verified against the running container**, not assumed. The host has no git, so it
was proven by content on 2026-07-31: in-container md5sums of `/app/graph_mcp/__main__.py`
(`98583ce9…`) and `/app/graph_mcp/service.py` (`dd2929ee…`) matched `git show 6dc3999:` for both
(recorded in IADC-Advisor commit `8069b17`). Reuse that method — comparing file content — whenever
the deployed sha needs establishing.

The one residual: **a graph image deployed after 2026-07-31 would leave this mirror lagging**, which
the ordering rule below permits and which is the harmless direction. Re-verify by the same method
before a release that matters.

For context on what the lag currently costs: at the time of mirroring, IADC-Core `HEAD` was ahead by
four of the seven files, but documented **the same 18 tools**. The difference is prose refinement, not
a missing tool — which is precisely the failure the ordering rule guards against.

---

## The ordering rule — it is release-blocking

**The skill may lag the deployed server, never lead it.** A server tool the skill does not mention
is harmless. A skill promising a tool the deployed server lacks makes Claude call it and fail.

So: **deploy the graph image first, then refresh this mirror from the sha that built it, then
publish.** Never refresh from IADC-Core `HEAD` — `HEAD` can be ahead of what is deployed, which is
the harmful direction. At the time of writing, `HEAD` *is* ahead: four of the seven files differ.

Refresh is triggered by a graph **deploy**, not by a release schedule.

---

## Refreshing

Mechanical. Nothing here may be hand-edited — a fix belongs upstream in IADC-Core, where a
drift-guard test couples the skill to the server's real tool roster on every commit.

```bash
# from IADC-Core, with <sha> = the sha that built the newly deployed graph image
git archive <sha> .claude/skills/iadc-graph \
  | tar -x -C ../IADC-Marketplace/iadc-graph --strip-components=3
```

Then verify the copy is clean and update the table above:

```bash
# byte-identity, ignoring the manifest this repo adds
diff -r --exclude=.claude-plugin \
  <(git -C ../IADC-Core show <sha>:.claude/skills/iadc-graph >/dev/null; echo) /dev/null >/dev/null
diff -r --exclude=.claude-plugin iadc-graph <path-to-IADC-Core-worktree-at-sha>/.claude/skills/iadc-graph
```

The second command must print nothing.

## Why a copy at all

A `git-subdir` source pointing straight at IADC-Core would need no copy and would carry a true
`sha` pin — but every installer would then need git read access to IADC-Core, i.e. to the
review-tool source. See the family's
[ADR 0003](https://github.com/teamignyte/IADC/blob/main/docs/adr/0003-shared-skills-ship-as-pinned-marketplace-plugins.md).

Because the mirror is a **relative-path** plugin source rather than a git source, there is no `sha`
field in its marketplace entry to pin. The pin *is* this file: the copy plus the sha recorded above.
