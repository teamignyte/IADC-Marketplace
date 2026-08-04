---
name: setup
description: Configure the iadc-graph plugin's MCP connection for this repo — collect the graph URL and API key, write the `iadc` entry into `.mcp.json` (merging, never overwriting, preserving every other server), and run the git-safety sequence a credential needs before any value is written. Designed as the one place in the family that writes this entry, so other IADC plugins can point here instead of carrying their own copy. Skips itself when a working entry already exists. Works with neither other IADC plugin installed. Run once per repo, before the graph is needed.
disable-model-invocation: true
---

# Setup

`iadc-advisor` and `iadc-tester` both reach the graph over one MCP server, `iadc`, configured in the
client repo's `.mcp.json` — a URL plus an `appian-api-key` header. Writing that entry means writing a
**credential**, which is why it lives here instead of in either product: `iadc-graph` is the one
plugin both already depend on, so this is the one place in the family the sequence below needs to
exist. Nothing here reads anything `iadc-advisor` or `iadc-tester` write, and nothing here assumes
either is installed — this works whether either, both, or neither is present.

This is prompt-driven, not a script: explore, show the user what you found, get their word before
any git change or credential write, then write. **Never print a credential value back into the
transcript.** Report `.mcp.json`'s *shape* — server names, key names, `••••` where a secret sits —
from the first time you read the file through the last time you show it, in every step below.

## Process

### 1. Check whether this repo already needs anything from you

Read `.mcp.json` if it exists (redacted, per the rule above; if it exists but doesn't parse, treat
that the same as "nothing confirmed yet" here — step 4 handles a malformed file explicitly when it's
actually time to write). Look at the `iadc` entry, if there is one:

- **No file, no `iadc` key, or its `url` / `headers.appian-api-key` are still `<placeholder>` text**
  — nothing usable yet. Continue to step 2.
- **Both values look literal** — check whether tools named `mcp__iadc__*` (e.g. `mcp__iadc__seed`)
  are available to you in this session — directly listed, or discoverable if your environment defers
  tool schemas. Claude Code connects the servers named in `.mcp.json` when a session starts, so their
  presence means this entry was already live at the start of *this* session:
  - **Present** → a working entry. Say so — name the `url`, redact the key — and stop. Nothing else
    in this skill needs to run.
  - **Absent** → on record, but not confirmed from here, and that is genuinely ambiguous rather than
    a "no": the entry could be wrong, or it could just be a value nothing has loaded yet, because a
    `.mcp.json` edit only takes effect on the *next* session — a value written five minutes ago in
    this same session would look identical to a wrong one from this vantage point. Say both readings
    plainly and ask: enter fresh values now (treat as unconfigured — continue to step 2), or leave it
    standing and check again after restarting? On "leave it," stop without writing anything.

This is the only evidence this skill gathers for "working," and it stops short of proving the key
itself is authorized. Every read tool needs a `session_id`, which only exists after `seed()` — which
either needs a server-local export path a remote client doesn't have, or kicks off a real Appian
export against a real application. That's not a lightweight ping to run just to check a connection,
and this skill never collects an application to run it against anyway (see the standalone note
above). So a `mcp__iadc__*` tool being present confirms the transport connected with the configured
header attached; it is evidence the URL is right and the server answered, not proof the key would be
accepted for actual graph work.

### 2. Establish the ignore rule — before anything is written

`git check-ignore .mcp.json`.

- **Succeeds** — already covered, by this line or a broader one already in place. Nothing to do;
  continue to step 3.
- **Fails** — show the user the line this needs and get an explicit yes before adding it:

  ```
  # iadc-graph credential — the iadc MCP entry in .mcp.json, never committed
  .mcp.json
  ```

  Create `.gitignore` if it doesn't exist yet. **On decline, add nothing, and don't ask again this
  run** — remember it: step 4 writes no credential value below, and hands the two values off instead
  of putting them in the file.

(If these commands fail because this repo has no `.git` yet, say so and stop there. The checks below
have nothing to run against, and writing a credential with no safety net in place is the one thing
this skill won't do.)

### 3. Handle a tracked `.mcp.json`

Before writing anything to the file itself — whichever way step 2 came out —
`git ls-files --error-unmatch .mcp.json`.

- **Fails** (untracked, or the file doesn't exist yet) — continue to step 4.
- **Succeeds** (tracked) — a tracked file is never ignored, whatever `.gitignore` says. Stop and
  surface it: someone committed this deliberately at some point, and it has to stop being tracked
  before a credential can land in it. Propose `git rm --cached .mcp.json`. Make **no** git change
  without an explicit yes.

  - **Decline** → this step ends here. Write **no** credential — leave any `<placeholder>` exactly as
    it is, and don't add an `iadc` block at all if there isn't one already. Tell the user the two
    values this entry still needs (`url`, `appian-api-key`) and that they need to live somewhere
    outside this repo until it's untracked. Stop; there is nothing else for this skill to do this run.
  - **Accept** → run it, then re-check `git check-ignore .mcp.json`:
    - **Succeeds** — untracked and ignored, but only in the **index**: `git rm --cached` touches the
      index and nothing else, so **HEAD still carries the file**. `git reset`, `git restore --staged
      .mcp.json`, or `git stash` would each put it straight back to tracked, where no `.gitignore`
      line reaches it and the next broad `git add` would stage the credential you're about to write.
      So this has to be **committed before any credential is written**. Show `git status` so the
      pending deletion is visible, and offer to commit it now — run that only on their yes.
      - **They commit now** → continue to step 4.
      - **They'd rather commit it themselves** → write **no** credential this run. Leave the
        placeholders standing, name the values still needed, and repeat the pending deletion in your
        final report so it isn't the one thing forgotten.
    - **Still fails** — step 2's line was never actually added (declined, or never reached). Write
      **no** credential: explain that `git rm --cached` only *staged* a deletion — the working-tree
      file is untouched — so the next broad `git add` would re-add whatever it currently holds as a
      fresh blob, which makes writing a credential right now actively worse, not safer. Then either
      settle step 2 now (this is the first ask since the file became untracked, not a re-nag of an
      earlier decline) and re-check, or, on a second decline, offer `git restore --staged .mcp.json`
      to put the repo back the way it was — run only on their yes — and stop.

### 4. Collect the values and write

First, the file itself has to be usable. `.mcp.json`:

- **Doesn't exist** — fine, there's nothing to merge into; skip to writing, below.
- **Exists, parses as JSON, and `mcpServers` is an object (or is simply absent)** — usable; continue.
- **Exists but doesn't parse, or parses to something where `mcpServers` isn't an object** — **stop.
  Write nothing.** Show the user the exact problem — the parse error, or which part of the shape is
  wrong — and let them choose: fix it and re-run this skill, or describe what the file should hold so
  you can rebuild it with their explicit confirmation before anything is written. Guessing a repair
  risks silently discarding whatever they had reason to put there.

Re-confirm `git check-ignore .mcp.json` succeeds before anything literal goes in — the last gate, and
the one that catches a step-2 decline on a repo where step 3 found no tracked file to react to (so
nothing upstream already stopped this run). If it still fails, write no credential and point back at
the missing line.

Now ask the user for the graph URL and the API key. See
[mcp-entry-template.json](./mcp-entry-template.json) for the exact shape to fill in, including the
default port and scheme. Fill in **literal values, not `${VAR}`** — the Windows Desktop app doesn't
reliably expand `${VAR}` in `.mcp.json`. Drop the template's `_comment` keys; they're notes to you,
never configuration, and must not appear in the file you write.

- **`.mcp.json` doesn't exist** — write a new file holding just `{"mcpServers": {"iadc": {...}}}`.
- **`.mcp.json` exists** — **merge**: add or update only `mcpServers.iadc`. Every other key, at the
  top level and inside `mcpServers`, is untouched — not just in content but in formatting and order,
  so edit the `iadc` block in place (or insert it) rather than parsing the whole file and
  reserializing it, which would restyle everything else along with it. Show the user the diff —
  redacted — before writing.

### 5. Report

Say plainly, for this run:

- The `.mcp.json` entry as it now stands — redacted — or that it was deliberately left untouched, and
  exactly which values are still needed and why (declined ignore rule, tracked file, malformed JSON).
- Any `git rm --cached` deletion still only **staged**, not committed — repeat this even if step 3
  already said it, so it isn't the one thing missed between steps.
- If a fresh value was written this run: it won't show as connected in *this* session — `.mcp.json`
  is read when a session starts, not while one is already running. Start a new session in this repo
  to pick it up; running `/iadc-graph:setup` again at that point reports it as already working
  (step 1) instead of asking again.
