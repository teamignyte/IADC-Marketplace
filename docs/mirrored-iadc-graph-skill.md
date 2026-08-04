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
`git log 5195a74..b461811 -- graph_mcp/ api/ evaluator/ graph/ graph_view/ graphify_adapter/
reader/ resolver/ sail/ .claude/skills/iadc-graph/` in IADC-Core — the full import closure derived
in the ordering rule below, not just `graph_mcp/` — returns only IV-375's two commits (the same
result the narrower filter this entry used to cite gave), so nothing this refresh carries documents
behaviour beyond what `5195a74` actually built and deployed.

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
this mirror lagging again, which the ordering rule below permits. A deploy log (this refresh's
method) shows a deploy happened, not what is running now — see the ordering rule's first step for
why that gap matters and when to close it by content (the `6dc3999` method) instead.

---

## The ordering rule — it is release-blocking

**The mirrored skill may lag the deployed server, never lead it.** A server tool the skill does not
mention is harmless. A skill promising a tool the deployed server lacks makes Claude call it and
fail.

So: **deploy the graph image first, then refresh this mirror from the sha that built it, then
publish.** Refreshing straight from IADC-Core `HEAD` is the harmful direction **unless** a check
establishes the deployed server has not moved. The check has three parts; skip any one, or any part
turns up a behaviour change, and the default stands: deploy first, refresh from the sha that built
it, publish.

**1. Get `<deploy-sha>`, and know what it does and doesn't prove.** A successful graph-image deploy
is recorded as a committed log under `.scratch/<feature>/results/` in IADC-Core — there is no single
fixed path or filename shape; naming is per-feature, and **most deploy logs name no sha at all** —
checked across six: four (`.scratch/epic216-arch/results/w2-deploy-dev.log`,
`.scratch/graph-filters-and-review-resend/results/04-deploy.log`,
`.scratch/iv320-portal/results/EPIC-deploy.log`,
`.scratch/arch-round3-split/results/10-deploy-graph-split.log`) carry no sha anywhere in filename or
content; only two do — `.scratch/arch-round3-split/results/27-deploy-5195a74.log` by filename, and
`.scratch/epic1-parity/results/10-deploy.log` by a header line
(`deploy start … — shipping main@b371734`). Take the newest sha-bearing log, across every
`.scratch/*/results/`, for a deploy that
built the **graph** image specifically (a portal- or review-only deploy proves nothing about the
graph server) — and when none names a sha, there is no `<deploy-sha>` to diff commits against, so
skip straight to content verification below.

**A deploy log proves a deploy happened, not what is running now** — a rollback, a hotfix, or a
redeploy from a different branch after the log was written all break that inference, silently, and
the log carries no signal that any of them occurred. When that distinction matters — before a
release that matters, or whenever no log names a sha — don't trust the log alone: re-verify by
content against the live container instead, the `6dc3999` method above (compare in-container file
content against `git show <candidate-sha>:<path>`; reach the host through IADC-Core's `iadc-ops`
skill).

**2. Check the deployed server's actual import closure, not just `graph_mcp/`.** `graph_mcp/`
imports first-party `api/`, `graph/`, `graphify_adapter/`, `reader/` and `resolver/` directly; those
in turn import `evaluator/`, `graph_view/` and `sail/`. Derived, not hand-listed, by walking every
module- and function-level `import`/`from` statement with `ast` (a deferred import still runs the
first time its code path executes) — run from IADC-Core's root:

```bash
python3 - <<'PY'
import ast, pathlib
ROOT = pathlib.Path(".")
FIRST_PARTY = sorted(p.name for p in ROOT.iterdir() if p.is_dir() and (p / "__init__.py").is_file())
def top(name): return name.split(".")[0]
edges = {p: set() for p in FIRST_PARTY}
for pkg in FIRST_PARTY:
    for f in (ROOT / pkg).rglob("*.py"):
        if "__pycache__" in f.parts:
            continue
        for node in ast.walk(ast.parse(f.read_text(), filename=str(f))):
            if isinstance(node, ast.Import):
                mods = [top(a.name) for a in node.names]
            elif isinstance(node, ast.ImportFrom) and not node.level and node.module:
                mods = [top(node.module)]
            else:
                mods = []
            edges[pkg].update(m for m in mods if m in FIRST_PARTY and m != pkg)
closure, frontier = {"graph_mcp"}, ["graph_mcp"]
while frontier:
    pkg = frontier.pop(0)
    for tgt in sorted(edges[pkg]):
        if tgt not in closure:
            closure.add(tgt); frontier.append(tgt)
print("first-party packages (dir with __init__.py):", ", ".join(FIRST_PARTY))
print("graph_mcp's import closure:              ", ", ".join(sorted(closure)))
print("excluded (no path into the closure):     ", ", ".join(sorted(set(FIRST_PARTY) - closure)))
PY
```

```
first-party packages (dir with __init__.py): api, evaluator, graph, graph_mcp, graph_view, graphify_adapter, harness, portal, reader, resolver, sail, tests
graph_mcp's import closure:               api, evaluator, graph, graph_mcp, graph_view, graphify_adapter, reader, resolver, sail
excluded (no path into the closure):      harness, portal, tests
```

So the check is `git log <deploy-sha>..HEAD -- graph_mcp/ api/ evaluator/ graph/ graph_view/
graphify_adapter/ reader/ resolver/ sail/ .claude/skills/iadc-graph/`, read every commit it lists,
don't just count them. `harness/`, `portal/` and `tests/` sit outside the closure and stay out of
the filter. **This closure is IADC-Core's shape today, not a permanent list.** When a new
first-party top-level package appears and anything already in the closure starts importing it,
re-run the command above over the current tree rather than adding a name by hand — a hand-maintained
list drifts the moment someone adds a package and no one remembers to update this doc; a re-derived
one doesn't, because it walks the tree instead of trusting memory.

**3. Read every listed commit for behaviour, not for file type.** "Touches only
`.claude/skills/iadc-graph/`" is a claim about *which directory changed*, not about *what the change
promises*. This doc's own case study above makes the point with a real split: of the four files
IADC-Core `HEAD` was ahead by at the time, one (`references/identifiers-and-discovery.md`) really
was harmless — a pure rename — but the other three (`SKILL.md`,
`references/return-shapes-and-errors.md`, `references/session-lifecycle.md`) carried the
`seed(export_ref=...)` constraint, the second error class, the guarded session string and the
background sweeper — none of it touching `graph_mcp/`, all of it promising something the running
server either does or doesn't actually do. File type alone can't tell those two groups apart; only
reading the diff can. Read what each listed commit's diff *says*; a behaviour promise anywhere in
it, code or prose, is a behaviour commit.

IV-375's refresh (`b461811`) is the worked example: the deploy log at
`.scratch/arch-round3-split/results/27-deploy-5195a74.log` fixes the deploy sha at `5195a74`
(filename convention: sha in the name), and the `git log` above, run over the full closure, lists
only the two prose-only commits this ticket made — neither promises anything the server doesn't
already do.

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
