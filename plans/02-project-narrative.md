# Stage 02 — Adopt Project Narrative

Source: `BrightFlagProxyMCPBuilder/prompts/02-adopt-project-narrative.md`

## Context

Project Narrative is installed *before* any decision-bearing stage, so that the choices made in
Stages 3–10 are captured as they are taken rather than reconstructed afterwards. It is a
deterministic, review-first decision-history mechanism — not a changelog generator — and it must
never invent rationale from code or diffs.

This stage is mechanical. Its only interesting property is that it cannot complete without human
action on GitHub, which makes it the one hard stop in the sequence.

## Preconditions

Stage 1 committed: solution builds clean, tests green, snapshot checked in.

## Scope in

Running the current installer, configuring the title through supported configuration, verifying with
`narrative check`, surfacing the manual follow-ups, documenting the two-PR lifecycle.

## Scope explicitly out

Everything else. No configuration, no fake server, no contracts. And specifically: no hand-editing
of `Narrative.md`, and no reconstruction of generated files from memory.

## Work items

### 1. Read the installer contract first

Read the current `install.md` in `jamiemitchellconsultants/Narrative` **before** editing anything.
The installer's contract is authoritative and may have moved; do not run from memory of a previous
project.

### 2. Run the installer

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative install
```

From the repository root. Node v26.5.0 is present.

### 3. Pull-request template

Do not overwrite an existing template. This repository has none, so the installer's template
applies. If one appears, add the three headings without changing their spelling:

- `## Narrative Context`
- `## Narrative Decision`
- `## Narrative Consequences`

### 4. Title

Set the Narrative title to `BrightFlag Proxy MCP Server Narrative` **only through supported
configuration** — the generated `Narrative.md` is a projection of the fragments, never a source.

### 5. Verify

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

Must return zero. Report the result honestly; do not weaken validation to get a pass.

### 6. Surface the manual follow-ups — do not claim them done

These are GitHub repository settings. I cannot perform them and must not report them as complete:

1. Enable read/write Actions workflow permissions, and allow GitHub Actions to create and approve
   pull requests.
2. Create the repository label named exactly `narrative-required`.
3. Commit and merge the scaffolded files to the default branch **before Stage 3**.

The installing pull request itself carries **no** `narrative-required` label. The workflow cannot
capture the PR that first installs it, because workflows must already exist on the default branch
for that to work.

### 7. Document the two-PR lifecycle

Record in the repository that these are decision-bearing changes, each requiring a
`narrative-required` PR with explicit Context, Decision, and Consequences:

- the fixed BrightFlag operation set;
- the definition of approved for payment;
- the approved-status allow-list;
- the lookback limit;
- the permitted payment statuses;
- the exposed MCP tool surface;
- the ontology contract.

Lifecycle: labelled project PR merges → automation opens a separate narrative proposal → review and
merge that proposal **without** the label, to avoid recursion.

## Tests

None beyond `narrative check`. This stage adds no code paths.

## Acceptance checks

- Configuration, preamble, generated narrative, workflows, and PR template all exist.
- `narrative check` returns zero.
- Workflows and template use the identical label spelling and section headings.
- No rationale was invented.
- The installation PR is unlabelled.
- The learner has been told, explicitly, to stop until the installation is published and merged.

## Stage boundary — this one is a hard stop

Commit the bootstrap locally. Suggested message: `Install Project Narrative`. Do not push
automatically.

Then **pause the sequence entirely** and report the three manual items above. Stage 3 does not begin
until you confirm the installation is published and merged to the default branch. This is not a
courtesy pause: the label and workflow permissions have to exist before any later stage's
`narrative-required` PR can behave correctly.

## Risks

- The installer reaches the network via `npx`. If that is unavailable, stop and report — do not
  hand-write the scaffold.
- Repository settings for Actions permissions may be restricted at the organisation level. If so,
  that is an org-admin action, not a repository one, and blocks the same gate.
- Should the installer's contract have changed since `install.md` was last read, follow the current
  contract, not this plan; note the divergence in the stage report.
