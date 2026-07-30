# BrightFlagProxyMCPBuilder — Claude Code Instructions

> Read this file in full before changing anything. Every rule here is binding — regardless of which
> AI tool is reading it. This file is the single source of truth; `AGENTS.md`, `GEMINI.md`,
> `.github/copilot-instructions.md`, `.cursor/rules/`, `.windsurf/rules/`, and `.clinerules/` are
> thin pointers back to it (for Codex/OpenCode, Gemini CLI, GitHub Copilot, Cursor, Windsurf, and
> Cline respectively) so the rules never have to be kept in sync across multiple copies. Edit this
> file, not those.

Before anything else, read [START-HERE.md](START-HERE.md), [README.md](README.md), and the **Index**
at the top of [Narrative.md](Narrative.md). The index is cheap to read every session; open a
specific entry's full text only when the task needs that entry's actual rationale. Also read
[DESIGN-CALLS.md](DESIGN-CALLS.md) before revisiting any judgement call it records.

---

## §1 — What this repository is

This repo teaches the construction of a capability-limited BrightFlag invoice-payment MCP server. It
contains **prompts and plans, not the server**. The implementation lives elsewhere.

- `prompts/` is authoritative. It defines each stage's requirements.
- `plans/` describes *how* a prompt is applied to a concrete implementation repository.
- **Where a plan and its prompt disagree, the prompt wins**, and the divergence is a defect in the
  plan, not a permitted variation.

Every plan follows one structure: Context, Preconditions, Scope in, Scope explicitly out, Work
items, Tests mapped one-to-one onto the prompt's own "Prove" list, Acceptance checks with runnable
commands, Stage boundary, and Risks. Every plan ends by naming what must **not** begin next, because
the sequence's value comes from stopping at each boundary. Preserve that shape.

A change that adds, removes, or renumbers a stage must update `prompts/`, the matching `plans/`
file, and the table plus open-decisions list in `plans/README.md` in the same change. A stage that
is contingent — not part of the version 1 sequence — must say so in the prompt, the plan, and the
index, because a reader who implements a contingent stage merely because it is numbered can do real
harm.

---

## §2 — Narrative discipline

This repo treats the history of decisions as a primary artifact. Every session that makes a
meaningful decision — about the taught architecture, the prompt sequence, governance, or a
correction to an earlier decision — records an entry. This is not optional documentation; it is the
mechanism this repo is testing.

`Narrative.md` is a **generated file**. Its own first line says so. Never hand-edit it or its Index.

### The label and the pull-request body are both required

Entries are produced by the [Project Narrative][pn] action, wired up in `.github/workflows/`. On
merge it reads the pull request and proposes a fragment on a separate draft pull request. It needs
**two** things, and fails or silently does nothing if either is absent:

1. The `narrative-required` label, applied **before merge**.
2. Three headings in the pull-request **body**, named exactly:
   - `## Narrative Context`
   - `## Narrative Decision`
   - `## Narrative Consequences`

`.github/pull_request_template.md` already contains these. **Do not bypass the template.** Creating
a pull request with `gh pr create --body ...` replaces the template wholesale, which is the single
easiest way to lose a narrative entry — it has already happened. If you pass `--body`, carry the
three sections in it yourself.

Missing label: the workflow exits quietly and no entry is ever produced. Missing sections with the
label present: the workflow **fails visibly**. Neither is recoverable after the fact — the workflow
triggers on the merge event only, so labelling a merged pull request does nothing. A missed entry
has to be written by hand as a fragment.

Do **not** apply `narrative-required` to a narrative-only pull request (a proposed fragment, or a
repair like this one), because that would recursively create an entry about maintaining the
narrative.

### Writing a fragment by hand

Only needed when an entry was missed, or when correcting the record. Create
`narrative/entries/YYYYMMDD-<slug>.md`:

- Front matter, required: `date` (`YYYY-MM-DD`), `slug` (lower-case kebab-case, and the filename
  after the date prefix must match it exactly), `title`, `summary` (respecting
  `summaryMaxCharacters` in `.project-narrative.json` — currently 240), `kind`, `status`.
- Optional: `sequence`, a full-precision UTC instant that orders same-day entries in true merge
  order; and `evidence`, the pull-request URL plus merge commit.
- `kind` is one of `architecture`, `product`, `governance`, `operational`, `correction`,
  `experiment`. `status` is one of `proposed`, `accepted`, `superseded`.
- Body: exactly `## Context`, `## Decision`, `## Consequences`, in that order, none empty. Note the
  plural on the last one; the validator rejects `## Consequence`.

Then **recompile and commit the output**:

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative compile
```

In this repo `Narrative.md` **is** committed alongside its fragments — `validate-narrative.yml` runs
`check`, which fails when the compiled output is stale. (Other repositories in this family invert
this and have CI recompile on `main`; do not carry that habit across.)

### Never rewrite an accepted entry

An accepted entry records what was decided and why **at the time**. When a later decision refines or
reverses it, add a new entry with `kind: correction` that links back by slug — do not edit the
original so it reads as though the better framing had been there all along. That would destroy the
evidence that the framing ever needed correcting, which is the one thing this repo is trying to
keep.

Cite entries by slug (`#entry-<slug>`), never by number. Numbers are positional and shift as entries
are added.

---

## §3 — Git and review discipline

- Branch names follow `category/short-name` — `decision/`, `fix/`, `clarify/`, `contingency/`,
  `narrative/`, `automation/`.
- **Do not stack a pull request on another pull request's branch.** If the base merges first, the
  stacked branch is orphaned: GitHub reports it as merged while its commits never reach `main`. This
  has happened twice. Branch from `main` and accept a textual reference to an unmerged decision.
- After pushing follow-up commits to a branch with an open pull request, say so explicitly. A pull
  request merged before later commits arrive silently drops them, and the merge looks clean.
- Verify what actually landed with `git log origin/main --oneline` after a merge, not by trusting
  the pull request's state.
- Commit or push only when asked. Never force-push a shared branch or delete a remote branch unless
  explicitly requested.

## §4 — Prose conventions

- Prompts and plans wrap at 100 columns.
- No emoji.
- Use absolute dates (`YYYY-MM-DD`), never relative ones.
- State limitations plainly rather than qualifying them away. A dev deployment proves nothing about
  a live topology; a retention window is not a guarantee; an untested variant is not supported. The
  sequence's credibility rests on this.
- Do not describe a skipped check as passing. If something cannot be executed in the current
  environment, label it a manual gate with the exact command and expected result.

[pn]: https://github.com/jamiemitchellconsultants/Narrative
