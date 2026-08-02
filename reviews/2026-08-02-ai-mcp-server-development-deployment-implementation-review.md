# Implementation review — `ai-mcp-server` development deployment

Reviewed plan: [2026-08-02-ai-mcp-server-development-deployment-plan.md][plan]

Date: 2026-08-02

## Scope inspected

- `BrightFlagProxyMCPBuilder` on `decision/ai-mcp-server-development-deployment`, baseline
  `7bfe8e8fb9818f92fc7b8cc2e14fc10c32faf9f1`, including the generated Stage 18 and 19 prompts
  and plans.
- `LocalAI` on `feat/brightflag-mcp-deployment`, baseline
  `2120a8fedafa34717e9de9a10fa610eb06aab571`, including the pending BrightFlag Windows
  deployment changes.
- The approved plan and the implemented Prompt 17 server baseline where needed to test sequencing
  and contract assumptions.

This document preserves the twelve findings presented during implementation review and the
operator's disposition of each. It records the review; it does not itself redefine the approved
requirements.

## Disposition summary

| # | Finding | Disposition |
|---:|---|---|
| 1 | Stage 18 moved to the Stage 19 Keycloak topology too early | Corrected plan-first |
| 2 | Deployment guessed a customer-specific approved-invoice status | Corrected plan-first |
| 3 | Production-origin exclusion cannot be enforced from known data | Skipped by instruction |
| 4 | Keycloak mode could deploy without completed Keycloak setup | Deferred to LocalAI #28 |
| 5 | Existing Keycloak mappers were accepted by name alone | Deferred to LocalAI #28 |
| 6 | Deployment state was saved before health was established | Accepted for this home lab |
| 7 | GitHub token was accepted as a command argument | Corrected in LocalAI |
| 8 | Caddy validation and reload were optional success conditions | Corrected in LocalAI |
| 9 | Fixed-token materialisation depended on the selected auth mode | Deferred to LocalAI #28 |
| 10 | Rollback used a mutable tag, not the recorded ID | Accepted for this home lab |
| 11 | The device-token manual gate required an impossible claim set | Open; not actioned |
| 12 | Stop and removal could report success after command failure | Corrected in LocalAI |

## Findings

### 1 — Stage 18 changed the Keycloak topology before Stage 19

The generated Stage 18 prompt and plan described the shared `homelab` realm and `mcp-client` even
though Stage 18 starts from Prompt 17's dedicated `brightflag-mcp` realm and pre-registered public
client. Stage 19 owns the later host-deployment migration to the shared realm and caller client.
Applying Stage 18 as written would therefore either change Prompt 17 outside the intended stage
boundary or depend on topology that Stage 19 had not introduced.

**Disposition:** correct the approved review plan first, then regenerate the Stage 18 and 19 prompt
and plan wording. Stage 18 now preserves Prompt 17's contract; Stage 19 alone performs the shared
realm/client migration.

### 2 — Deployment guessed the approved-invoice status

`setup-brightflag-mcp-windows.ps1` defaulted `ApprovedInvoiceStatus` to `APPROVED FOR PAYMENT`.
BrightFlag status vocabularies are customer-specific, and the builder sequence requires reviewed
tenant evidence rather than a plausible label. The default could silently select no invoices or
the wrong invoices while appearing configured.

**Disposition:** correct the approved review plan first, regenerate Prompt and Plan 19, then fix
LocalAI. Deployment now requires the reviewed integration-test tenant status explicitly, defines no
default, and records the value for rollback.

### 3 — The production-origin exclusion is not enforceable from this host's knowledge

`BrightFlagOrigin` is configurable while the documents state that a production BrightFlag URL must
not be introduced. The home-lab deployment knows the approved integration-test default, but it has
no authoritative inventory of BrightFlag production URLs against which an arbitrary supplied URL
could be rejected. A negative repository grep cannot prove an operator-supplied endpoint is not
production.

**Disposition:** skipped by operator instruction. This deployment has no knowledge of the
BrightFlag production URL, and no guessed production-deny list was added.

### 4 — Keycloak mode could continue without functioning Keycloak configuration

`Set-KeycloakConfiguration` warned and returned when the Keycloak Compose file, environment file,
or bootstrap password was absent. It also warned and continued when the configured user did not
exist. The script could then start `-AuthMode keycloak` even though it had not established a caller
capable of receiving the required roles and claims.

**Disposition:** defer a complete, fail-closed, dedicated Keycloak setup to
[LocalAI issue #28][localai-28].

### 5 — Existing Keycloak mappers were accepted by name alone

`Add-ClientMapper` treated a matching mapper name as sufficient and left it unchanged. A mapper
with the expected name but the wrong protocol mapper, claim name, audience, value, type,
multivalued setting, or access-token setting would therefore pass setup while producing tokens the
server refuses or interprets incorrectly.

**Disposition:** defer exact desired-state creation and mapper read-back verification to
[LocalAI issue #28][localai-28] with finding 4.

### 6 — Deployment state was saved before health was established

The deployment wrote `deployment-state.json` immediately after `docker compose up` and before the
health loop completed. A container that starts but never becomes healthy is consequently recorded
as the current deployment and can displace the previously recorded rollback relationship.

**Disposition:** accepted without change for this monitored home-lab deployment.

### 7 — The GitHub credential was exposed as a command parameter

The script accepted `-GitHubToken` in addition to `MCP_GITHUB_TOKEN`. Supplying a secret in the
invocation can expose it through shell history and process arguments, contradicting the script's
own claim that the credential does not enter either surface.

**Disposition:** corrected in LocalAI. The command parameter was removed, credentials are read only
from `MCP_GITHUB_TOKEN`, and repository URLs containing user information are rejected.

### 8 — Caddy validation and reload were optional

`Write-CaddyFragment` could create an ingress directory when the shared ingress was absent, warn
when Caddy was stopped, and continue after a reload failure. Deployment could therefore report
success even though the canonical endpoint was not serving the new configuration.

**Disposition:** corrected in LocalAI. A present and running shared ingress is now a precondition;
validation and reload failures restore the previous fragment and stop deployment.

### 9 — Fixed-token materialisation depended on the selected auth mode

The script created and materialised the fixed MCP token only when `AuthMode` was `fixed`, while the
approved plan described three managed secret identifiers and an explicit rotation lifecycle. A
first deployment in Keycloak mode therefore did not establish the fixed-token secret or make its
rotation lifecycle independent of the currently selected authentication mode.

**Disposition:** defer the auth-independent fixed-token lifecycle to
[LocalAI issue #28][localai-28] with findings 4 and 5. Current documentation now states the actual
conditional behaviour instead of claiming all three secrets always exist.

### 10 — Rollback selected a mutable image tag rather than the recorded image ID

Deployment state recorded both the previous image tag and image ID, but rollback verified and used
only the tag. A later build or retag could move that tag to different content while the retained
image ID remained available, so rollback was not bound to the exact recorded image identity.

**Disposition:** accepted without change for this manually watched home-lab deployment.

### 11 — The device-token manual gate required an impossible claim set

The Stage 19 plan said a decoded Keycloak device-flow token must carry *only* `iss`, `aud`, `tid`,
and flat `roles`. Keycloak access tokens necessarily include additional standard and protocol
claims such as subject and lifetime data. The useful assertion is that the four named claims exist
with the required values, not that no other claim exists.

**Disposition:** open. No action was requested for finding 11 in this review pass, so the manual
gate remains unchanged.

### 12 — Stop and removal could report false success

`Invoke-StopOnly` did not inspect the result of `docker compose stop`. `Invoke-RemoveOnly` did not
fail on `docker compose down`, Caddy validation, or Caddy reload errors and could still print its
success messages. Removal also reported success when neither owned artifact existed.

**Disposition:** corrected in LocalAI. Both paths now preflight their dependencies, check native
command exit codes, restore the fragment on validation or reload failure, and refuse an empty
removal rather than claiming work was completed.

[plan]: 2026-08-02-ai-mcp-server-development-deployment-plan.md
[localai-28]: https://github.com/jamiemitchellconsultants/LocalAI/issues/28
