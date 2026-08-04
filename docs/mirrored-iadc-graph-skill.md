# The mirrored `iadc-graph` skill — source sha and how to refresh it

`iadc-graph/skills/iadc-graph/` in this repo is a **copy** of the canonical skill in
[IADC-Core](https://github.com/teamignyte/IADC-Core) (`.claude/skills/iadc-graph/`), taken at the
sha that built the **deployed** graph image — or, when the ordering rule below's check establishes
the deployed server hasn't moved, at `HEAD` directly. It carries **zero local patches**. The enclosing
`iadc-graph/` plugin directory additionally holds `.claude-plugin/plugin.json`, which makes it
installable.

| | |
|---|---|
| **Upstream** | `teamignyte/IADC-Core`, path `.claude/skills/iadc-graph/` — *ours*, not a third party |
| **Mirrored at** | `1f85211` (IADC-Core `iv-356-per-project-config-parity`, not yet on `main`) — taken 2026-08-04 |
| **Contents** | `iadc-graph/skills/iadc-graph/` holds 7 `.md` files: `SKILL.md` + 6 under `references/` |
| **Local patches** | **none in `iadc-graph/skills/iadc-graph/`.** Any difference from upstream there is staleness — take upstream |

**This row owes a re-record — a merge prerequisite of `iv-356-per-project-config-parity`, not of
any one ticket on it.** `1f85211` lives on that same unpushed feature branch — unresolvable from
any other clone, and dead the moment the branch is rebased or squashed. **Whoever merges
`iv-356-per-project-config-parity` to `main` must re-record this row at the resulting `main` sha
before publishing again** — the obligation carries forward from `b461811` (IV-375's pin) to
`1f85211` (IV-381's pin) unchanged; it is a property of the branch, not of whichever commit
happens to be pinned when a reader looks. Until the merge, treat this pin as provisional.

`1f85211` is IV-381's fix for this skill's own violation of the epic's rule that a plugin may read
only config it writes itself: `SKILL.md` named `docs/agents/advisor.md` — a file only Advisor
writes — as the per-project config source, and fell back to the Appian MCP to resolve an
application UUID when no such file existed, the exact server IV-364 removed from Tester-only
installs. One commit, no `graph_mcp/` edit. Verified the same way `b461811` was: against
IADC-Core's own deploy log, `.scratch/arch-round3-split/results/27-deploy-5195a74.log` records a
successful graph-image deploy (`DEPLOY_EXIT=0`, `graph /health` responding) at `5195a74`, dated
2026-08-03, still the newest graph-deploy log and still an ancestor of `1f85211`.
`git log 5195a74..1f85211 -- graph_mcp/ api/ evaluator/ graph/ graph_view/ graphify_adapter/
reader/ resolver/ sail/ vendor/graphify/graphify/ .claude/skills/iadc-graph/` in IADC-Core — the
full filter derived in the ordering rule below, not just `graph_mcp/` — returns three commits:
`5ef7e16`/`b461811` (IV-375, already read and cleared by this entry's prior revision) plus
`1f85211` itself (IV-381) — a `SKILL.md`-only prose change that removes an instruction rather than
adding one, touches no `graph_mcp/` path, and promises no tool or behaviour the deployed server
doesn't already have. So nothing this refresh carries documents behaviour beyond what `5195a74`
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

**1. Get `<deploy-sha>`, and know what it does and doesn't prove.** Take the newest deploy log under
`.scratch/*/results/` in IADC-Core for a deploy that built the **graph** image specifically —
marked `Image iadc-graph:latest Building` post-split, or `Image iadc:latest Building` for the
pre-split monolith that served graph too (a portal- or review-only deploy proves nothing about the
graph server) — and read `<deploy-sha>` from it. Naming is per-feature, not one fixed path or
filename shape, and **most deploy logs name no sha at all**: checked across six, only two do —
`.scratch/arch-round3-split/results/27-deploy-5195a74.log` (both by filename and by its own header
line, `# DEPLOY of main=5195a74`) and `.scratch/epic1-parity/results/10-deploy.log` (a header line
only, `deploy start … — shipping main@b371734`, no sha in the filename); the other four carry no sha
anywhere. **When the newest graph-deploy log is one of those four, there is no `<deploy-sha>` to diff
commits against — go straight to content verification, next paragraph, to get one.**

**A deploy log proves a deploy happened, not what is running now** — a rollback, a hotfix, or a
redeploy from a different branch after the log was written all break that inference, silently, and
the log carries no signal that any of them occurred. When that distinction matters — before a release
that matters, or whenever no log names a sha — don't trust the log alone: pick a **candidate** sha
(your best guess at what's live — typically the newest commit on `main` you have independent reason
to believe was deployed) and confirm it by content instead of by log, the `6dc3999` method above
(compare in-container file content against `git show <candidate-sha>:<path>`; reach the host through
IADC-Core's `iadc-ops` skill). **Once content-confirmed, that candidate sha IS `<deploy-sha>`** — feed
it into parts 2 and 3 below. Content verification supplies the sha the commit-by-commit check needs
when no log does; it does not replace that check.

**2. Check `graph_mcp`'s package closure — a safe superset of the deployed image, not the image
itself — plus the one dependency that closure can't structurally see.** `graph_mcp/` imports
first-party `api/`, `graph/`, `graphify_adapter/`, `reader/` and `resolver/` directly; those in turn
import `evaluator/`, `graph_view/` and `sail/`. Derived, not hand-listed, by walking every module- and
function-level `import`/`from` statement with `ast` (a deferred import still runs the first time its
code path executes) — run from IADC-Core's root, and **re-run before every refresh, unconditionally**:
there's no judgment call to make about whether a package "probably" joined since the run itself is
cheap (2.4s measured, `time python3 …` this session).

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
IADC-Core b461811, 2026-08-04:
first-party packages (dir with __init__.py): api, evaluator, graph, graph_mcp, graph_view, graphify_adapter, harness, portal, reader, resolver, sail, tests
graph_mcp's import closure:               api, evaluator, graph, graph_mcp, graph_view, graphify_adapter, reader, resolver, sail
excluded (no path into the closure):      harness, portal, tests
```

This is `graph_mcp`'s repo-wide **package** closure, not the deployed image's own closure — a
narrower, already-**enforced** answer to that exists: `tests/test_image_closure.py`'s
`test_graph_entrypoint_closure_never_touches_review_only_modules` (:351) and
`test_graph_stage_chain_provisions_every_non_api_package_it_imports` (:496) probe
`graph_mcp.service`'s real runtime imports in a fresh subprocess and check them against the
Dockerfile's actual `COPY` lines, per commit — and that probe proves the graph image excludes
`evaluator/` (`Dockerfile:158`, review-target-only) and five of `api/`'s thirteen files
(`contract.py`, `contract_pull.py`, `main.py`, `models.py`, `pipeline.py`; `Dockerfile:152-153`,
also review-target-only — the `graph` target COPYs only the eight shared files at
`Dockerfile:128-130`). Consult that test directly for the precise answer;
this doc's filter deliberately stays at the wider package closure regardless, because
**over-inclusion here only costs more commits to read, which is safe — the asymmetry that matters is
under-inclusion.**

Under-inclusion is exactly where the closure above fails, structurally.
`graphify_adapter/render.py:73` does `from graphify.build import build_from_json` — the vendored
`vendor/graphify/graphify/` (PyPI `graphifyy`), which `Dockerfile:112` COPYs into both images and
which `test_graph_stage_chain_provisions_every_non_api_package_it_imports` (:496) checks. **The
script above cannot see this edge**: `vendor/` has no `__init__.py` at IADC-Core's root, so it's
never in `FIRST_PARTY`, and the import name (`graphify`) doesn't match the directory name (`vendor`)
regardless. A manual `grep -rn 'import vendor\|from vendor'` across the closure — the cross-check
this derivation actually ran — returns nothing and reads as confirmation; it was checking the wrong
name for a real dependency. **Add `vendor/graphify/graphify/` to the filter by hand** — it is
vendored *third-party* source, not a first-party package, but a change to it is still a change in
what the deployed graph server runs, and nothing here derives it automatically.

That this is a real gap, not a theoretical one — the old (narrow) filter passes silently over a
commit that changes what the deployed graph server actually does, and adding
`vendor/graphify/graphify/` catches it:

```bash
$ git log ce0b121^..ce0b121 --oneline -- graph_mcp/ .claude/skills/iadc-graph/
$ # (nothing — the old filter passes this range silently)
$ git log ce0b121^..ce0b121 --oneline -- graph_mcp/ .claude/skills/iadc-graph/ vendor/graphify/graphify/
ce0b121 fix(vendor/graphify): revert the .sail delta now the shim is gone
```

`ce0b121` reverts a real `.sail`-registration delta in `vendor/graphify/graphify/detect.py` and
`extract.py` — exactly the class of change this filter exists to catch, and exactly what the narrow
filter would have missed.

So the check is:

```bash
git log <deploy-sha>..HEAD -- graph_mcp/ api/ evaluator/ graph/ graph_view/ graphify_adapter/ reader/ resolver/ sail/ vendor/graphify/graphify/ .claude/skills/iadc-graph/
```

Read every commit it lists, don't just count them. `harness/`, `portal/` and `tests/` sit outside the
closure and stay out of the filter. **The package-closure part of this list is IADC-Core's shape
today, not a permanent list** — when a new first-party top-level package appears and anything already
in the closure starts importing it, the unconditional re-run above picks it up automatically, no hand
edit needed. `vendor/graphify/graphify/` will **not** be picked up automatically by anything — this
file is the only place it's recorded — so a future vendored dependency of the same shape needs the
same by-hand addition and the same kind of note.

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
`.scratch/arch-round3-split/results/27-deploy-5195a74.log` fixes the deploy sha at `5195a74` (both by
filename and by its own header line), and the `git log` above, run over the full filter including
`vendor/graphify/graphify/`, lists only the two prose-only commits this ticket made. Both were read
in full: neither promises anything the server doesn't already do.

Refresh is triggered by a graph **deploy**, not by a release schedule.

---

## Refreshing

Mechanical. Nothing in `iadc-graph/skills/iadc-graph/` may be hand-edited — a fix belongs upstream
in IADC-Core, where a drift-guard test couples the skill to the server's real tool roster on every
commit.

```bash
# from IADC-Core, with <sha> = the sha that built the newly deployed graph image, OR the sha at
# HEAD when the ordering rule above's check established the deployed server hasn't moved
git archive <sha> .claude/skills/iadc-graph \
  | tar -x -C ../IADC-Marketplace/iadc-graph/skills --strip-components=2
```

Then verify the copy is clean and update the table above — if `<sha>` came from a permitted `HEAD`
refresh rather than a deploy, record the deploy-sha the check was run against too (e.g. "taken at
HEAD, check run against deploy-sha `<X>`"), so a later reader can tell which of the two paths earned
this pin:

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
