# Prompt 2 — Adopt Project Narrative

Using the reusable contract, implement Stage 2: install Project Narrative before decisions about
approved-payment evidence, payment writes, and ontology reporting are implemented.

Project Narrative is a deterministic, review-first decision-history mechanism. It is not a changelog
generator and must never invent rationale from code or diffs.

## Read and run the current installer

Read the current `INSTALL.md` in `jamiemitchellconsultants/Narrative` before editing. From the root
of the consumer repository, run the installer exactly as its current contract specifies:

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative install
```

Do not reconstruct generated files from memory and do not overwrite an existing pull-request
template. If a template already exists, add the three required headings without changing their
spelling:

- `## Narrative Context`
- `## Narrative Decision`
- `## Narrative Consequences`

Set the Narrative title to `BrightFlag Proxy MCP Server Narrative` only through supported
configuration. Do not hand-edit `Narrative.md`; fragments are authoritative and the compiled file is
a projection.

Run:

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

Report the result honestly. Do not weaken validation.

## Bootstrap and repository actions

This installation is mechanical and must not carry the `narrative-required` label. The workflow
cannot capture the pull request that first installs it because workflows must already exist on the
default branch.

Surface the installer's manual follow-ups exactly:

1. Enable read/write Actions workflow permissions and allow GitHub Actions to create and approve
   pull requests.
2. Create the repository label named exactly `narrative-required`.
3. Commit and merge the scaffolded files to the default branch before Prompt 3.

Do not report any repository setting or label as completed unless you actually verified or changed
it through an authorized repository action.

Document the two-pull-request lifecycle and make clear that later changes to the fixed BrightFlag
operations, the definition of approved for payment, the approved-status allow-list, the lookback
limit, the permitted payment statuses, the exposed MCP tools, or the ontology contract are
decision-bearing:

1. A meaningful project pull request carries `narrative-required` and explicit Context, Decision,
   and Consequences.
2. After it merges, automation opens a separate narrative proposal.
3. Review and merge that proposal without `narrative-required` to avoid recursion.

## Agent instruction files

The narrative lifecycle above only works if whichever coding agent runs a later stage actually knows
about it. Do not leave that to the agent's defaults.

Write `CLAUDE.md` at the implementation repository root as the **single source of truth** for agent
instructions, and state in it that it is binding regardless of which tool reads it. It must cover,
at minimum: that `Narrative.md` is generated and must never be hand-edited; that a decision-bearing
pull request needs both the `narrative-required` label and the three headings
`## Narrative Context`, `## Narrative Decision` and `## Narrative Consequences` **in the
pull-request body**; that the
maintenance workflow fires on the merge event only, so neither omission can be repaired afterwards
by labelling; that a narrative-only pull request must not carry the label; and that an accepted
entry is never rewritten — a later reversal is a new entry of kind `correction` citing the original
by slug.

State plainly that creating a pull request with a supplied body replaces the repository template
wholesale, and that doing so without carrying the three sections forward is the most common way an
entry is silently lost.

Then add **thin pointer files** for the other tier-one agents, each of which says only that
`CLAUDE.md` is authoritative, that every rule there is binding, and that instruction changes go in
`CLAUDE.md` rather than in the pointer:

- `AGENTS.md` — Codex, OpenCode, and any agent reading the generic file;
- `.github/copilot-instructions.md` — GitHub Copilot;
- `GEMINI.md` — Gemini CLI;
- `.cursor/rules/claude-instructions.mdc` — Cursor, with `alwaysApply: true` front matter;
- `.windsurf/rules/claude-instructions.md` — Windsurf, with `trigger: always_on` front matter;
- `.clinerules/claude-instructions.md` — Cline.

The pointers must **not** restate the rules. Duplicated instructions drift, and a stale copy is
worse than no copy because an agent cannot tell which is current. Do not list a pointer location the
repository does not actually contain — a pointer to an absent directory teaches a future reader that
the set is maintained when it is not.

This file set is mechanical scaffolding and is part of the unlabelled installation change.

## Acceptance criteria

- Configuration, preamble, generated narrative, workflows, and pull-request template exist.
- `narrative check` returns zero.
- Workflows and template use the exact same label and section headings.
- No narrative rationale was invented.
- `CLAUDE.md` exists and states the label rule, the three required pull-request body headings, the
  merge-event-only limitation, and the never-rewrite-an-accepted-entry rule.
- Every pointer file exists, names `CLAUDE.md` as authoritative, restates none of its rules, and
  references no location the repository does not contain.
- The installation pull request is unlabelled.
- The learner is explicitly told to stop until the installation is published and merged.

Commit the bootstrap locally with a focused message. Do not push automatically. Pause the learning
sequence until the learner explicitly publishes and merges the installation.
