# Stage 19L — Restore LocalAI for Prompt 20

Source: `BrightFlagProxyMCPBuilder/prompts/19l-restore-localai-for-prompt-20.md`

## Context

Prompt 19 transfers BrightFlag deployment ownership to LocalAI and deliberately installs a
plaintext, per-service-audience deployment. LocalAI also contains independently reviewed shared-host
and other-MCP preparation for Prompt 20. Replaying Prompt 19 therefore leaves the server sequence
correct at Stage 19 but rewinds BrightFlag-specific LocalAI values that Stage 20 assumes already
exist.

Stage 19L is the cross-repository reconciliation. It restores only the Prompt 20 BrightFlag values
inside LocalAI, preserves all independent work, and leaves Prompt 20 as a server-only stage.

## Preconditions

- Prompts 18 and 19 are applied and green in `BrightFlagProxyMCPServer`.
- Prompt 19's LocalAI deployment changes have been applied or merged.
- Current LocalAI `main` contains the independently prepared shared `homelab-mcp` realm model,
  dynamic registration, Caddy issuer alias, and the other MCP server's deployment work.
- Work begins from current LocalAI `main` on a new branch with LocalAI's binding instructions read.
- Any pre-existing working-tree change is inventoried and preserved before editing.

## Scope in

Surgical reconciliation of the BrightFlag deployment script; BrightFlag-specific README and
Keycloak-guide wording; a new LocalAI narrative entry; public HTTPS, shared audience, trusted proxy,
read-only Keycloak preflight, Caddy automatic HTTPS, removal of the private matcher and host-gateway
entry; and verification that every surviving Prompt 18 and 19 control remains.

## Scope explicitly out

Checking out historical files wholesale; reverting a merge or independent LocalAI work; editing the
shared host script, ontology deployment, another MCP server, either BrightFlag repository, root
Caddyfile, shared Keycloak or LocalStack topology; live deployment; Prompt 20 server implementation;
production BrightFlag configuration; and contingent Stages 12 or 13.

## Work items

### 1. Inventory independent LocalAI work

Read the current instructions, narrative, branch history, and affected files. Identify the semantic
Prompt 19 changes rather than treating a prior commit as a tree to restore. Record any overlap with
independent work and stop if it requires an unprovided design choice.

### 2. Verify the shared-host prerequisite without editing it

Confirm the current tree owns the `homelab` realm, `homelab-mcp` audience, realm-default audience
and constant-claim scope, capability scopes, public fallback client, dynamic registration,
external `mcp-public`, Caddy fragment import, and the `auth.tqaentry.com` Caddy network alias. A
missing item is a failed precondition, not permission to add it to the BrightFlag script.

### 3. Restore BrightFlag's Prompt 20 deployment values

Make the script generate the HTTPS resource, `homelab-mcp` audience, explicit Keycloak mode,
trusted-proxy declaration, internal HTTP listener, three unchanged allowed hosts, bare Caddy site,
and non-buffering reverse proxy. Remove the private-source matcher and container host-gateway entry.

### 4. Return Keycloak ownership to the shared host

Remove BrightFlag's resource client, roles, mappers, designated-user grants, administrator secret
path, and `kcadm` writes. Keep only a read-only discovery preflight against the canonical realm.

### 5. Preserve fixed mode and Prompt 19's surviving controls

Default to Keycloak without deleting or warning against fixed mode. Check both exclusive environment
branches, both role values, explicit status, three allowed hosts, exact-commit build, source label,
state, retained rollback image, three LocalStack identifiers, protected file materialisation,
redaction, unpublished port, external network, integration-test boundary, and deliberate hardening
omissions.

### 6. Reconcile documentation and narrative

Update only current BrightFlag and shared-audience wording. Preserve the other MCP server and shared
host documentation. Prepend a LocalAI narrative entry that supersedes the current Prompt 19
BrightFlag posture without deleting its historical entry.

## Tests

1. Diff the branch against its merge base and assert only the expected BrightFlag script,
   documentation, and narrative files changed.
2. Parse `docs/setup-brightflag-mcp-windows.ps1` with PowerShell's AST parser and fail on any parser
   error.
3. Assert the Keycloak branch's mode, shared audience, HTTPS resource URI, trusted-proxy posture,
   proxy name, and configured `brightflag.read` scope.
4. Assert the fixed branch still emits every fixed-token setting and both roles, with no obsolete
   unsupported-mode warning.
5. Assert the generated Compose contract retains one unpublished service, external `mcp-public`,
   exactly three allowed hosts, explicit approved status, active-mode file secrets, all three
   configurable secret identifiers, and both roles.
6. Assert the Caddy template is a bare hostname with `flush_interval -1`, with no private matcher.
7. Assert no BrightFlag `host-gateway` entry and verify the shared-host Caddy alias source instead.
8. Assert the BrightFlag script contains no Keycloak administrative write and retains its read-only
   discovery preflight.
9. Assert exact-commit build, labelled revision, deployment state, previous image, rollback,
   transient credential, LocalStack materialisation, ACL, and redaction paths still exist.
10. Grep current documentation for the former HTTP endpoint and per-service audience; permit them
    only in historical narrative.
11. Run LocalAI's repository checks and prove no shared-host, ontology, other-MCP, or unrelated file
    changed.

## Acceptance checks

```powershell
$tokens = $null
$errors = $null
[System.Management.Automation.Language.Parser]::ParseFile(
    (Resolve-Path 'docs/setup-brightflag-mcp-windows.ps1'),
    [ref]$tokens,
    [ref]$errors) > $null
if ($errors.Count) { $errors | ForEach-Object { Write-Error $_ }; exit 1 }
```

```bash
git diff --check
rg -n "homelab-mcp|https://brightflag-mcp\.tqaentry\.com/mcp|TrustedProxy|Caddy ingress" \
  docs/setup-brightflag-mcp-windows.ps1 README.md docs/configure-keycloak.md
```

```bash
if rg -n "http://brightflag-mcp\.tqaentry\.com/mcp" \
  docs/setup-brightflag-mcp-windows.ps1 README.md docs/configure-keycloak.md; then
  exit 1
fi
if rg -n "remote_ip private_ranges|auth\.tqaentry\.com:host-gateway" \
  docs/setup-brightflag-mcp-windows.ps1; then
  exit 1
fi
```

Inspect `git diff --name-only "$(git merge-base HEAD main)"` and account for every path. LocalAI's
own checks remain authoritative and must also pass.

Manual gates, not run by this stage: render Compose and the Caddy fragment on the Windows host;
validate the whole Caddy configuration; prove certificate issuance and public HTTPS health; retrieve
Keycloak discovery and JWKS from inside the container without `host-gateway`; complete dynamic
registration and authorization-code with PKCE from a conversational agent; exercise device flow and
fixed mode; and prove read and payment in both modes. Record exact commands, expected results,
absolute date, and operator when those gates are run.

## Stage boundary

Commit locally under LocalAI's narrative rules. Do not push unless requested. Return to the
BrightFlag server repository before applying Prompt 20. Do not deploy, mutate live shared services,
edit either server from LocalAI, or begin Prompt 20 on this branch.

## Risks

- A whole-file restore can silently discard the independent Prompt 20 and other-MCP preparation
  this stage exists to preserve.
- Restoring HTTPS without the shared Caddy alias leaves Keycloak discovery or JWKS unreachable from
  the container; verify the prerequisite instead of reintroducing `host-gateway`.
- Restoring the old Prompt 20 script verbatim can reintroduce two stale defects: omitting explicit
  `CallerIdentity__Mode=Keycloak` and warning that fixed-token mode is unsupported after Prompt 18.
- Removing fixed mode because Keycloak becomes the default violates Prompt 20's explicit retention
  of Prompt 18.
- A partial LocalAI/server rollout either fails HTTPS resource validation or refuses every token on
  audience. Play Stage 19L before Stage 20 and deploy them together only after both are reviewed.
