# The `iadc-graph` skill mirror has moved

`iadc-graph` used to be mirrored **in this repo**, at `iadc-graph/skills/iadc-graph/` — a **copy**
of the canonical skill authored in [IADC-Core](https://github.com/teamignyte/IADC-Core)
(`.claude/skills/iadc-graph/`), taken at the sha that built the **deployed** graph image — or, when
the maintainer procedure's check established the deployed server hadn't moved, at `HEAD` directly.
It shipped alongside `.claude-plugin/plugin.json` and the hand-written `skills/setup/` in that same
`iadc-graph/` subtree.

**As of IV-398/IV-399, the whole plugin — mirror, manifest and setup skill — lives in its own repo,
[IADC-Graph-Plugin](https://github.com/teamignyte/IADC-Graph-Plugin)**, root layout (`skills/…`,
not nested under an enclosing `iadc-graph/` directory the way this repo's copy was). This repo has
no test suite and no CI, so the hand-written `setup` skill had nowhere to be checked; the whole
plugin moved to a dedicated client-facing repo that has both.

`IADC-Marketplace`'s catalogue entry for `iadc-graph` in `.claude-plugin/marketplace.json` is now a
pointer at that repo — the same `github`-source shape `iadc-tester`'s entry already uses. There is
no local copy left here for anyone to read or hand-edit.

That entry carries no `ref` or `sha`, so an install takes `IADC-Graph-Plugin`'s default-branch tip
at install time. The ordering rule below is therefore enforced on that repo's `main`: refreshing its
mirror ships to every new install immediately, with no second gate in this repo.

The refresh procedure, the current pinned IADC-Core sha, and the ordering rule (the mirror may lag
the deployed graph server, never lead it) are a maintainer concern that lives in IADC-Core's
`docs/marketplace-mirror-refresh.md`, not in this client-facing repo.

## Why a copy at all

A `git-subdir` source pointing straight at IADC-Core would need no copy and would carry a true
`sha` pin — but every installer would then need git read access to IADC-Core, i.e. to the
review-tool source. See the family's
[ADR 0003](https://github.com/teamignyte/IADC/blob/main/docs/adr/0003-shared-skills-ship-as-pinned-marketplace-plugins.md).
