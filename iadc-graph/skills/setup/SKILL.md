---
name: setup
description: Configure the iadc-graph plugin's MCP connection for this repo — collect the graph URL and API key, write the `iadc` entry into `.mcp.json` (merging, never overwriting, preserving every other server), and run the git-safety sequence a credential needs before any value is written. Designed as the one place in the family that writes this entry, so other IADC plugins can point here instead of carrying their own copy. Skips itself when a working entry already exists. Works with neither other IADC plugin installed. Run once per repo, before the graph is needed.
disable-model-invocation: true
---

# Setup

`iadc-advisor` and `iadc-tester` both reach the graph over one MCP server, `iadc`, configured in the
client repo's `.mcp.json` (repo root) — a URL plus an `appian-api-key` header. Writing that entry
means writing a **credential**, which is why it lives here instead of in either product: `iadc-graph`
is the one plugin both already depend on, so this is the one place in the family the sequence below
needs to exist. Nothing here reads anything `iadc-advisor` or `iadc-tester` write, and nothing here
assumes either is installed — this works whether either, both, or neither is present.

This is prompt-driven, not a script: explore, show the user what you found, get their word before
any git change or credential write, then write. **Never print a credential value back into the
transcript.** Report `.mcp.json`'s *shape* — server names, key names, `••••` where a secret sits —
from the first time you read the file through the last time you show it, in every step below.

**Every gate below tests what git actually did, not what the user said or what a looser check
implies.** `git check-ignore` alone, for instance, goes green the instant an untracking is merely
*staged* — before it's committed — because it already accounts for the index, not for whether HEAD
still carries the file. Where that distinction matters, the steps below say which command settles it
and why the other one doesn't.

## Process

### 1. Check whether this repo already needs anything from you

Read `.mcp.json` at the repo root if it exists (redacted, per the rule above; if it exists but
doesn't parse, treat that the same as "nothing confirmed yet" here — step 4 handles a malformed file
explicitly when it's actually time to write). Look at the `iadc` entry, if there is one:

- **No file, no `iadc` key, or its `url` / `headers.appian-api-key` are still `<placeholder>` text**
  — nothing usable yet. Continue to step 2.
- **Both values look literal** — check whether tools named `mcp__iadc__*` (e.g. `mcp__iadc__seed`)
  are available to you in this session — directly listed, or discoverable if your environment defers
  tool schemas. Claude Code connects the servers named in `.mcp.json` when a session starts, so their
  presence means this entry was already live at the start of *this* session:
  - **Absent** → on record, but not confirmed from here, and that is genuinely ambiguous rather than
    a "no": the entry could be wrong, or it could just be a value nothing has loaded yet, because a
    `.mcp.json` edit only takes effect on the *next* session — a value written five minutes ago in
    this same session would look identical to a wrong one from this vantage point. Say both readings
    plainly and ask: enter fresh values now (treat as unconfigured — continue to step 2), or leave it
    standing and check again after restarting? On "leave it," stop without writing anything.
  - **Present** → before calling this "working," check `git ls-files --error-unmatch .mcp.json`
    (repo root) — a connection succeeding is not the same question as a credential being safe:
    - **Untracked** → genuinely working. Say so — name the `url`, redact the key — and stop. Nothing
      else in this skill needs to run.
    - **Tracked** → **don't report this as working.** The key is committed into this repo's git
      history, and untracking the file later doesn't erase that — anyone with the repo's history can
      still read the old commit. Say so, then continue to step 2 exactly as the unconfigured case
      would: step 3's tracked-file handling is what gets *this* file untracked, and step 4 is where
      you'll ask whether to keep the value that's already there (it does work) or replace it outright
      — worth raising given the history point above. Don't skip ahead on the assumption nothing needs
      asking.

This tool-presence check is the only evidence this skill gathers for "working" — but it settles the
one question AC 4 needs settled: whether this specific credential is accepted by the graph service,
not whether every later operation will succeed. Every read tool needs a `session_id`, which only
exists after `seed()`, so this skill never calls one just to check a connection — and it never
collects an application to call one against anyway (see the standalone note above). Tool presence is
enough on its own, though: the graph service gates its entire MCP endpoint behind this exact key
before any MCP handshake completes, and does so fail-closed — a missing or wrong key can't match by
accident. So `mcp__iadc__*` tools being present doesn't just mean the transport connected; it means
this credential specifically was checked and accepted. What it doesn't prove: that a later `seed()`
or graph operation will succeed — those depend on things this check never touches (an application to
point at, whether Appian/LCP itself is reachable) — only that this credential is the right one.

### 2. Establish the ignore rule — before anything is written

Check whether `.gitignore` at the repo root already contains this exact line — read the file's
content directly for this, rather than `git check-ignore`: `check-ignore` reports "not ignored" for a
path that's currently tracked (or merely staged) even when the line is sitting right there in
`.gitignore`, so relying on it here risks proposing — and appending — a duplicate on a repo where
`.mcp.json` is already tracked with the rule already in place.

- **Line already present** — nothing to append. Continue below to the durability check.
- **Line missing** — show the user the line and get an explicit yes before adding it:

  ```
  # iadc-graph credential — the iadc MCP entry in .mcp.json, never committed
  .mcp.json
  ```

  Create `.gitignore` if it doesn't exist yet. **On decline, add nothing, and don't ask again this
  run** — remember it: step 4 writes no credential value below. Leave any `<placeholder>` exactly as
  it is, don't add an `iadc` block if there isn't one already, and hand the two values off instead of
  putting them in the file.

**Either way — whether the line was just added or was already sitting there — confirm it's actually
committed, not just present in the working tree or the index:** `git show HEAD:.gitignore` (redirect
a missing-file error) checked for the exact line. A line that's only a working-tree or staged change
is exactly as revertible as the `.mcp.json` removal step 3 guards against — a `git stash`, `git
checkout -- .gitignore`, or similar puts it back to not covering `.mcp.json` at all, and the next
broad `git add` then stages whatever the file holds.

- **Committed** — continue to step 3.
- **Not committed** — stage and commit just that file: `git add .gitignore`, then `git commit -m
  "Ignore .mcp.json — iadc-graph:setup" -- .gitignore` (this sequence is safe whether `.gitignore`
  already existed or is brand new, and never touches any other file's staged state, since it only
  ever adds and commits that one path). Show the pending change first, run only on an explicit yes.
  - **They'd rather commit it themselves** — write no credential this run either, for the same
    not-yet-durable reason. Name the values still needed.

(If these commands fail because this repo has no `.git` yet, say so and stop there. The checks below
have nothing to run against, and writing a credential with no safety net in place is the one thing
this skill won't do.)

### 3. Handle a tracked `.mcp.json`

Before writing anything to the file itself: `git ls-files --error-unmatch .mcp.json` (repo root).

- **Fails** (untracked, or the file doesn't exist yet) — continue to step 4.
- **Succeeds** (something is in the index at this path) — figure out which of two situations this
  is, since they need different explanations: `git cat-file -e HEAD:.mcp.json`.

  - **Succeeds (HEAD carries a committed blob)** — the ordinary case: someone committed this
    deliberately at some point. Say so, propose `git rm --cached .mcp.json`. Make **no** git change
    without an explicit yes.
    - **Decline** → this step ends here. Write **no** credential — leave any `<placeholder>` exactly
      as it is, and don't add an `iadc` block at all if there isn't one already. Tell the user the two
      values this entry still needs (`url`, `appian-api-key`) and that they need to live somewhere
      outside this repo until it's untracked. Stop; there is nothing else for this skill to do this
      run.
    - **Accept** → run it. `git rm --cached` touches only the **index** — HEAD is untouched — so
      re-confirm: `git cat-file -e HEAD:.mcp.json` (still succeeds; expected, and exactly the gap that
      matters). This is **not yet durable**: `git reset`, `git restore --staged .mcp.json`, or `git
      stash` would each put the index straight back in sync with HEAD — tracked again, where no
      `.gitignore` line reaches it and the next broad `git add` would stage the credential about to be
      written. **Don't treat `git check-ignore` succeeding here as clearance** — it goes green the
      moment the index changes, staged or committed, so it cannot tell these two states apart; only
      HEAD moving does. So this has to be **committed before anything is written**. Show `git status`
      so the pending deletion is visible. Before offering to commit, check what else is currently
      staged (`git diff --cached --name-only`):
      - **Nothing else** — offer a plain `git commit -m "Stop tracking .mcp.json —
        iadc-graph:setup"`. Run only on yes.
      - **Something else is staged too** — name it, and ask plainly: commit everything staged
        together now (only on an explicit yes naming what's included), or hold off?
      - **They commit now** → re-confirm `git cat-file -e HEAD:.mcp.json` now fails — verify the
        effect, not the attempt — then continue to step 4.
      - **They'd rather commit it themselves, or decline bundling unrelated staged work** → write
        **no** credential this run. Leave the placeholders standing, name the values still needed, and
        repeat the pending commit in your final report so it isn't the one thing forgotten.

  - **Fails (HEAD does not carry it)** — this path is staged but was **never committed**: there is no
    git history to untangle, but it still needs to come out of the index before anything can be
    safely ignored (a staged-only add is enough on its own to make `check-ignore` report "not
    ignored"). Say so plainly — don't reuse the "someone committed this deliberately" framing above;
    it would be false here. Propose the same `git rm --cached .mcp.json`, explicit yes required.
    - **Decline** → same as the decline branch above: no credential, placeholders left alone, name
      what's needed, stop.
    - **Accept** → run it. Nothing further is needed for this file specifically — there is no pending
      deletion to commit (confirm, don't assume: `git diff --cached --name-only` shows nothing for
      this path, and `git cat-file -e HEAD:.mcp.json` still fails). Continue directly to the gate
      below.

Once `.mcp.json` is confirmed untracked — either branch above, or step 3 was never entered because it
was untracked already — one gate remains: `git check-ignore .mcp.json`.

- **Succeeds** — continue to step 4.
- **Fails** — step 2's line either isn't there or isn't committed yet. Offer to settle step 2 now
  (this is the first ask since the file became untracked, not a re-nag of an earlier decline): on
  yes, go run step 2's whole sequence — append if missing, then confirm it's committed — and once
  that succeeds, come back here and treat this gate as passed; the destination is step 4, nothing
  vaguer than that. On a second decline: stop, write no credential, say what's still needed.

### 4. Collect the values and write

First, the file itself has to be usable. `.mcp.json` at the repo root:

- **Doesn't exist** — fine, there's nothing to merge into; the other two triage bullets below don't
  apply, so move on to the gates that follow this list — not straight to writing.
- **Exists, parses as JSON, and `mcpServers` is an object, or is simply absent** — usable; continue.
  An absent `mcpServers` isn't a problem to solve, just a key to create holding `iadc` alone.
- **Exists but doesn't parse, or parses to something where `mcpServers` isn't an object** — **stop.
  Write nothing.** Show the user the exact problem — the parse error, or which part of the shape is
  wrong — and let them choose: fix it and re-run this skill, or describe what the file should hold so
  you can rebuild it with their explicit confirmation before anything is written. Guessing a repair
  risks silently discarding whatever they had reason to put there.

**Three gates, all required, right before anything literal goes in — they test different things and
none alone is sufficient:**

- `git check-ignore .mcp.json` succeeds. Catches a step-2 decline on a repo where step 3 found no
  tracked file to react to (so nothing upstream already stopped this run). If it fails, write no
  credential and point back at the missing line.
- `git show HEAD:.gitignore` (redirecting a missing-file error), piped through a check for the exact
  line — `git show HEAD:.gitignore 2>/dev/null | grep -qxF '.mcp.json'` — **exits 0.** The pass
  condition is that whole pipeline's exit code, not just whether `git show` itself finds a
  `.gitignore` at HEAD: HEAD can carry a `.gitignore` that never contained this line — committed for
  some other rule entirely, with `.mcp.json` added only to the working tree afterward — and `git
  show` alone would still exit 0 there. This is step 2's own durability requirement, re-checked here
  rather than trusted from back there — `check-ignore` reads whatever `.gitignore` currently holds in
  the working tree, committed or not, so it stays green straight through step 2's "they'd rather
  commit it themselves" decline, which leaves the line only staged, or only in the working tree, and
  says "write no credential" without anything mechanical behind that sentence. This gate is what
  actually stands behind it. If the pipeline's exit code is nonzero, don't write — offer to settle it
  the way step 2 does (stage and commit just `.gitignore`, only on an explicit yes), and if that's
  declined too, stop for this run.
- `git cat-file -e HEAD:.mcp.json` **fails.** This is the one it's tempting to skip, because step 3
  usually already ensures it — but "usually" is the gap: a *prior*
  run of this same skill that got interrupted between untracking and committing leaves the file out
  of the index already, so step 3's own opening check (`ls-files`) sees nothing to react to and falls
  straight through here, while HEAD still carries the file. If this gate fails, don't write — say so,
  show `git status`, and offer to commit the removal now (checking first whether anything else is
  staged, exactly as step 3's commit offer does) before trying again. Skipping this because step 3
  "already handled it" is exactly the mistake this gate exists to catch.

Now ask the user for the graph URL and the API key — unless step 1 already found a literal,
connectable value on file (the tracked-but-working case), in which case say what's already there and
ask whether to keep it or replace it: a value that was ever committed can still be read from this
repo's history even after this run untracks the file, so replacing it is worth raising even though
it isn't required. See [mcp-entry-template.json](./mcp-entry-template.json) for the exact shape to
fill in, including the default port and scheme. Fill in **literal values, not `${VAR}`** — the
Windows Desktop app doesn't reliably expand `${VAR}` in `.mcp.json`. Drop the template's `_comment`
keys; they're notes to you, never configuration, and must not appear in the file you write.

- **`.mcp.json` doesn't exist** — write a new file holding just `{"mcpServers": {"iadc": {...}}}`.
- **`.mcp.json` exists, no `mcpServers` key yet** — add one, holding just `iadc`.
- **`.mcp.json` exists with a `mcpServers` object** — **merge**: add or update only
  `mcpServers.iadc`. Every other key, at the top level and inside `mcpServers`, is untouched — not
  just in content but in formatting and order, so edit the `iadc` block in place (or insert it)
  rather than parsing the whole file and reserializing it, which would restyle everything else along
  with it. Show the user the diff — redacted — before writing.

**After writing, read the file back and confirm it still parses as JSON, with the `iadc` block
holding what you just wrote.** An in-place text edit into hand-formatted JSON is exactly the kind of
change a stray comma slips into unnoticed, and a broken `.mcp.json` fails every server it lists —
`appian`, `context7`, `iadc` alike — not just this one. If it doesn't parse, say so immediately
rather than reporting success, and show what's wrong.

### 5. Report

Say plainly, for this run:

- The `.mcp.json` entry as it now stands — redacted — or that it was deliberately left untouched, and
  exactly which values are still needed and why (declined ignore rule, tracked file, malformed JSON).
- Any `.gitignore` line or `.mcp.json` untracking still only **staged or otherwise uncommitted** —
  repeat this even if an earlier step already said it, so it isn't the one thing missed between
  steps.
- If a fresh value was written this run: it won't show as connected in *this* session — `.mcp.json`
  is read when a session starts, not while one is already running. Start a new session in this repo
  to pick it up; running `/iadc-graph:setup` again at that point reports it as already working
  (step 1) instead of asking again.
