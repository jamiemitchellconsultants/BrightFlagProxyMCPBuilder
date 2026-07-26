# Prompt 2 — Adopt Project Narrative

Using the reusable contract, implement Stage 2: install Project Narrative before decisions about
approved-payment evidence, payment writes, and ontology reporting are implemented.

Project Narrative is a deterministic, review-first decision-history mechanism. It is not a
changelog generator and must never invent rationale from code or diffs.

## Read and run the current installer

Read the current `install.md` in `jamiemitchellconsultants/Narrative` before editing. From the root
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

## Acceptance criteria

- Configuration, preamble, generated narrative, workflows, and pull-request template exist.
- `narrative check` returns zero.
- Workflows and template use the exact same label and section headings.
- No narrative rationale was invented.
- The installation pull request is unlabelled.
- The learner is explicitly told to stop until the installation is published and merged.

Commit the bootstrap locally with a focused message. Do not push automatically. Pause the learning
sequence until the learner explicitly publishes and merges the installation.
