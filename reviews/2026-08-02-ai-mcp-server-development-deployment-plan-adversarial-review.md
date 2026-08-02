# Adversarial review — `ai-mcp-server` development deployment plan

Reviewed document: [2026-08-02-ai-mcp-server-development-deployment-plan.md][plan]

Date: 2026-08-02

Status: findings only. No corrections were implemented, and no file, issue, repository, or external
system outside this review document was changed.

## Baselines inspected

- `BrightFlagProxyMCPBuilder` — `5179a25` on `decision/local-pull-deployment`. Prompt 17 is merged
  to `origin/main` at `b6e7d19`.
- `BrightFlagProxyMCPServer` — `origin/main` at `6ee020d`. Prompt 17 is implemented and merged; the
  local checkout sits on `stage-16/local-pull-deployment` at `28fb914`, three commits behind
  `origin/main` and fully contained in it.
- `LocalAI` — `main` at `5209120`, clean working tree.
- `OntologyServerBuilder` — prompts `00`–`31`, inspected only.
- `OntologyService` — inspected only.

Findings are numbered by severity. Nothing below asks for production hardening, HTTPS, restricted
fixed-token access, LocalStack isolation, or the rewriting of an earlier prompt.

---

## Blockers

### B1 — Keycloak mode cannot authenticate: the `tid` and flat `roles` claim mappers are omitted

**Evidence.** Plan lines 196–208 enumerate the Keycloak desired state: realm, resource client,
`mcp-client`, device authorization, audience mapper, two roles, role assignment, no secret. No claim
mappers.

The server requires two claims Keycloak does not emit:

- `BrightFlagProxyMCPServer/src/BrightFlagMcp.Api/Configuration/CallerIdentityOptions.cs:105-113` —
  `Subject = "sub"`, `Tenant = "tid"`, `Roles = "roles"` (a flat array).
- Prompt 17's implemented deployment sets `RequiredClaims__0: tid` and `RequiredClaims__1: roles`
  (`BrightFlagProxyMCPServer/deploy/local/compose.yaml:40-41`), and the tenant value is checked
  against `BrightFlag__Tenant__Name` (`compose.yaml:34`); a mismatch is
  `CallerRejection.TenantMismatch`.
- Prompt 17's own manual gate already names this:
  `BrightFlagProxyMCPServer/deploy/local/manual-gates.md` §3 — "Inspect the realm, exact OpenCode
  redirect URI, PKCE S256, audience mapper, `tid`, flat `roles`, and the assigned user."
- LocalAI states the opposite shape as fact: `LocalAI/docs/configure-keycloak.md` §2.4 ("Realm roles
  surface as `realm_access.roles`; client roles as `resource_access.<client>.roles`") and
  `LocalAI/docs/ontology-keycloak.md:64` ("**Roles are not a flat `roles` array.**"). There is no
  `tid` anywhere in the `homelab` realm.

**Why it matters.** As specified, every Keycloak-issued token is refused at `RequiredClaimMissing`
before any role check runs. The Keycloak half of the plan does not work, and the manual gate at plan
line 269 ("Keycloak device flow grants read and payment access") would fail on first execution.

**Correction.** Add to Workstream 6 a hardcoded-claim mapper emitting `tid` with the exact
configured tenant value, and a mapper projecting the `brightflag-mcp` client roles into a flat
top-level `roles` array. Alternatively state in Prompt 18/19 that the server gains Keycloak-shaped
role and tenant reading (`realm_access` / `resource_access[...]`, tenant from reviewed configuration
rather than a token claim) and name the configuration keys. Add a manual gate that decodes a real
device-flow token and shows `iss`, `aud`, `tid`, and flat `roles`.

**Blocks implementation:** yes, for Keycloak mode.

### B2 — Prompt 17's implemented deployment stack stays active and contradicts Stage 19

**Evidence.** Prompt 17 is fully implemented and merged. The server repository ships a complete
rival deployment owner:

- `deploy/local/scripts/` — `Deploy-Local.ps1`, `Configure-Keycloak.ps1`,
  `Install-CaddyFragment.ps1`, `Materialize-Secrets.ps1`, `Preflight.ps1`, `Rollback.ps1`,
  `Stop.ps1`, `Validate.ps1`, `Set-LocalStackSecret.ps1`, `Collect-Diagnostics.ps1`,
  `Deployment.psm1`;
- `deploy/local/.env.example:5` — GHCR digest only; `:11-13` — `EDGE_NETWORK=edge_net` and
  `C:/mcp/edge`; `:10` — `MCP_RESOURCE_URI=https://.../mcp`; `:24-25` — realm `brightflag-mcp`;
- `deploy/local/scripts/Preflight.ps1:57-70` — fails closed unless LocalStack is loopback-only and a
  LAN-refusal evidence file exists;
- `deploy/local/compose.yaml:54` — only `ReadApprovedInvoicesRoles` is set; `MarkInvoicePaidRoles`
  is absent, which is how payment is disabled;
- `deploy/local/caddy/brightflag-mcp.caddy.template`, `docs/deploy-local.md`, and
  `deploy/local/manual-gates.md`.

The plan's supersession table (lines 112–124) names nine decisions and none of these artifacts.
Workstream 3's requirement list (lines 126–149) never requires retiring or rewriting them. Plan
lines 55–57 forbid changing the server repository in this work.

**Why it matters.** After Stage 19 the repository ships two mutually exclusive deployment owners,
and the server's own preflight fails closed on exactly the LocalStack posture Stage 19 accepts (plan
line 119). This is the precise failure the plan sets as its own test at lines 287–288 ("without
leaving mutually active artifacts or contradictory documentation"). It is also the most likely route
to an accidental direct edit of `BrightFlagProxyMCPServer`, because an implementer will find these
files contradicting the new prompt.

**Correction.** Add a supersession row — "`deploy/local` in the server repo is the deployment owner"
becomes "LocalAI `setup-brightflag-mcp-windows.ps1` is the sole owner" — and a Prompt 19 work item
naming each artifact and stating whether Prompt 19 deletes it, rewrites it, or marks it superseded
and inert, delivered by the additive prompt rather than by editing the server directly. Verification
item at plan line 261 ("no new diffs ... in the three read-only repositories") must then be
reconciled with the fact that Prompt 19, when later played, is supposed to change them.

**Blocks implementation:** yes.

### B3 — Several repository checks assert properties this plan cannot produce or test

**Evidence.** Plan lines 248–261 mix three incompatible check classes under one heading. These four
are properties of `BrightFlagProxyMCPServer`, which this work must not change (lines 55–57), and
which has no code in the builder repository:

- line 256 — "the model contains one BrightFlag container, no published backend port, and
  `mcp-public`". Ambiguous as well: the server's `deploy/local/compose.yaml` says `edge_net`, and
  the LocalAI-generated Compose is a different file;
- line 257 — "auth modes are mutually exclusive";
- line 258 — "fixed-token mode has no environment/profile restriction";
- line 259 — "the explicit plaintext mode accepts the HTTP external resource URI without claiming
  HTTPS".

**Why it matters.** As written these cannot pass at the end of this plan's work, because the code
that would satisfy them is produced by a later application of Prompts 18 and 19 to the server
repository. An acceptance list that cannot be satisfied is either abandoned or satisfied by editing
the forbidden repository. This also violates CLAUDE.md §4 — "Do not describe a skipped check as
passing."

**Correction.** Split the verification section into three: builder-repository checks on prompt and
plan text, structure, pairing, wrapping, and narrative compilation; LocalAI checks — PowerShell
parse, generated Compose free of placeholders and secret values, generated Compose declaring
`mcp-public` and no published port; and checks deferred to the future implementation of Prompts 18
and 19 in the server repository, each labelled with the stage that will run it and the exact
command.

**Blocks implementation:** yes. Acceptance is unachievable as stated.

---

## High

### H1 — Container-to-Keycloak issuer resolution is dropped without supersession

**Evidence.** Prompt 17 lines 99–101 require documenting and testing one exact split-DNS, hosts, or
host-gateway mechanism. It is implemented at
`BrightFlagProxyMCPServer/deploy/local/compose.yaml:27-28` as `extra_hosts:
"${KEYCLOAK_HOSTNAME}:host-gateway"`. The plan never mentions it — not in the table (lines 112–124),
not in Prompt 19's requirements (lines 126–149), not in the script responsibilities (lines 165–194).

LocalAI documents this exact trap and prescribes a different fix:
`LocalAI/docs/configure-keycloak.md` §5 — `KC_HOSTNAME` pins the issuer to the public name, so
discovery hands the container a public `jwks_uri`; the prescribed remedy is a `mcp-public` network
alias `auth.tqaentry.com` on the ingress Caddy, which is an edit to `setup-mcp-host-windows.ps1`.
That file is a shared service the BrightFlag script must not own (plan lines 185, 187–188) and is
not in the changeable-file list (lines 50–53).

The server's JWKS validation is unconditional on HTTPS — see, in `BrightFlagProxyMCPServer`,
`src/BrightFlagMcp.Api/Configuration/BrightFlagOptionsValidator.cs:489-499`, where plain HTTP is
permitted only for a loopback provider. No relaxation is available here, and none is wanted.

**Why it matters.** Without a named mechanism the container cannot fetch JWKS, and Keycloak mode
fails at first token validation regardless of B1.

**Correction.** Name one mechanism in Prompt 19 and in the LocalAI script: either carry
`extra_hosts: auth.tqaentry.com:host-gateway` into the generated Compose, which is self-contained
and owns nothing shared, or add `setup-mcp-host-windows.ps1` to the changeable-file list and require
the `mcp-public` alias there as an explicit reviewed change. Add a manual gate that runs the JWKS
fetch from inside the BrightFlag container.

**Blocks implementation:** yes, for Keycloak mode.

### H2 — The private-source matcher is claimed to reject Internet clients, and never tested

**Evidence.** Plan line 135 states as a Prompt 19 requirement "a private-source matcher that rejects
Internet clients". Line 27 states it as settled. Lines 279–280 then remove the proof: "External
proof that Caddy sees and rejects a genuinely non-private source is deferred to issue #45 and is not
a deployment acceptance gate."

Facts that make the claim doubtful rather than merely unproven:

- the router forwards inbound TCP 80 and 443 to this box
  (`LocalAI/docs/setup-mcp-host-windows.ps1:41-42, 66-71`), and firewall rules open 80/443 to
  `-Profile Any` (`:513-528`);
- the plan chooses `brightflag-mcp.tqaentry.com` in the same public zone as `auth.tqaentry.com` and
  `financeontology.tqaentry.com` but never states whether that name is a public A record or split
  DNS. Prompt 17 lines 110–113 required split DNS, and the plan's table does not supersede it;
- Docker Desktop on Windows publishes container ports through a host-side proxy. If the ingress
  Caddy container observes a Docker-internal source address for every client, `remote_ip
  private_ranges` matches everything and filters nothing — the matcher fails open, not closed. The
  same assumption already carries the Keycloak `/admin/*` restriction
  (`LocalAI/docs/setup-mcp-host-windows.ps1:380-384`), so it has never been isolated and tested
  either;
- Prompt 17's implementation already carries this as a live manual gate:
  `BrightFlagProxyMCPServer/deploy/local/manual-gates.md` §6.

**Why it matters.** The plan states a rejection property as fact while declining the one test that
distinguishes "configured" from "effective", which is what CLAUDE.md §4 forbids. If the matcher is
vacuous, a plaintext endpoint with full payment access is Internet-reachable — a materially
different situation from the accepted "plaintext bearer-token exposure on the physically controlled
LAN" (plan line 231). This is reported as a false claim and an unverifiable acceptance criterion,
not as a request to harden.

**Correction.** Either weaken lines 27 and 135 to "configures a private-source matcher whose
effectiveness under Docker Desktop port publishing is unproven, tracked by #45", or keep the claim
and restore the external-source check as a dated manual gate with the exact command and expected
result. In both cases state the DNS decision explicitly — public A record versus split DNS or a
hosts entry — since it determines whether the matcher is load-bearing at all.

**Blocks implementation:** no, but the claim as written is unsupported and must be corrected.

### H3 — Fixed-token mode adds a second authentication path the plan does not acknowledge

**Evidence.** The server's design invariant is explicit in code:
`BrightFlagProxyMCPServer/src/BrightFlagMcp.Api/Identity/CallerTokenValidator.cs:77-88` — "There is
exactly one of these, and both trust providers feed it: swapping the trust root swaps where keys and
an issuer come from and nothing else, so no environment can end up on a laxer path."
`CallerTrustProvider` is `None | Live | Local`
(`src/BrightFlagMcp.Api/Configuration/CallerIdentityOptions.cs:11-23`), and every accepted caller
passes signature, issuer, audience, lifetime, required-claim, and tenant-boundary checks in that one
type.

An opaque fixed token is not a JWT and cannot feed that validator. Plan lines 98–100 list the
consequences the prompt must state — shared identity, audit attribution, rate-limit bucket,
caller-bound plan scope — but omit that fixed-token mode bypasses the single validator entirely, and
with it the tenant-boundary check, expiry, and any revocation story.
`src/BrightFlagMcp.Api/Configuration/BrightFlagOptionsValidator.cs:338-360` also couples
`RequireCallerAuthentication` with `CallerTrustProvider`, which a third mode must change.

**Why it matters.** The plan's own review brief item 3 asks whether the absent production-profile
guard is implemented "without accidentally reintroducing fallback or partial access". That cannot be
answered without saying how the fixed-token path establishes `CallerIdentity`, the tenant boundary,
the rate-limit key, and the audit subject — all claim-derived today.

**Correction.** Prompt 18 should state that the fixed-token path does not traverse
`CallerTokenValidator`, name how each of those four values is established from reviewed
configuration instead, and require a test that exactly one branch is reachable per startup selection
rather than two branches with a precedence rule. Add the missing consequences — no expiry, no
revocation, no tenant claim corroboration — to the paragraph at lines 98–100.

**Blocks implementation:** no; materially incomplete.

### H4 — Audience exclusivity is unachievable with the shared `mcp-client`

**Evidence.** Plan lines 93–94 require "rejection of arbitrary `homelab` realm tokens that are not
addressed to `brightflag-mcp`". Lines 200–205 reuse "the existing public `mcp-client`, when
compatible" plus "an exact `brightflag-mcp` audience mapper".

In Keycloak the audience mapper lives on the caller client's dedicated scope and fires
unconditionally (`LocalAI/docs/configure-keycloak.md` §2.3), and `mcp-client` is designated the
shared caller for the ontology server too (§2.2). The device authorization grant carries no
`resource` parameter, so audience cannot be narrowed per request — Prompt 17 lines 86–88 relied on
the `resource` parameter with the authorization-code flow, which the plan replaces with device flow
(line 146) without noting the loss.

**Why it matters.** Once `brightflag-mcp` is added to `mcp-client`'s mappers, every token that
client mints — including ontology tokens — carries `aud: brightflag-mcp`. The audience check is then
not a boundary; only the role check separates the services, so the plan's stated rejection property
is false as written. Separately, ensuring `mcp-client`'s configuration is partial ownership of a
client shared with the pending `OntologyServerBuilder` prompt 32 work, which sits uneasily with plan
lines 187–188 and with the plan's framing that shared services are not taken over.

**Correction.** State plainly that with a shared caller client the audience is not a separator and
role assignment is the actual boundary, or give BrightFlag its own public caller client. Either way,
add "never remove or alter another service's mappers, scopes, or flows on `mcp-client`" to the
script's preserve list (plan line 187), and record the shared-client consequence against #46.

**Blocks implementation:** no if consciously accepted; the stated rejection property must be
corrected either way.

### H5 — The Git-ref workflow's cited model does none of what the plan attributes to it

**Evidence.** Plan lines 40–42: "resolves a configurable under-development Git ref once, builds that
exact commit, and records both values, following the approach in
`LocalAI/docs/setup-ontology-mcp-windows.ps1`". That script does none of it:

- `LocalAI/docs/setup-ontology-mcp-windows.ps1:225-233` builds `"$RepoUrl#$GitRef"` — the moving
  ref, resolved by BuildKit at build time;
- `:160-161` tags `ontology-service:$SafeRef` — ref-derived, not commit-derived. Two builds of the
  same branch overwrite one tag, so no previous image is retained;
- it records nothing, retains nothing, and has no rollback.

Also unaddressed:

- `BrightFlagProxyMCPServer/Dockerfile:24` and `:61` — `ARG SOURCE_REVISION=unknown`, surfaced as
  `org.opencontainers.image.revision` in the image label. The plan requires recording the requested
  ref, commit, and image identity (lines 139–140) but never says to pass `--build-arg
  SOURCE_REVISION=<commit>`, so the image would self-report `unknown`;
- the resolve mechanism is unspecified. `git ls-remote` is the obvious tool, and Git for Windows is
  a prerequisite of `LocalAI/docs/setup-mcp-host-windows.ps1:58` (checked at `:657`) but not of the
  cited ontology script. Plan item 2 (line 168) says only "transient repository authentication"; the
  naive form embeds the PAT in the URL, putting it in the process command line, which plan item 4
  (line 173) forbids only for LocalStack, Compose, generated files, and logs.

**Why it matters.** An implementer following the cited model verbatim satisfies none of plan
requirements 139–140 or 191–193, and the false attribution makes that outcome likely.

**Correction.** Delete or qualify "following the approach in `setup-ontology-mcp-windows.ps1`".
Specify the resolve command and how the credential reaches it without entering the process arguments
(`GIT_ASKPASS`, a credential helper, or `http.extraHeader` supplied via config rather than a URL);
require `--build-arg SOURCE_REVISION=$commit`; define the commit-derived tag format; and name the
record file holding requested ref, resolved commit, image tag, image ID, and previous deployable
image.

**Blocks implementation:** no; the claim is false and misleads implementation.

---

## Medium

### M1 — Host validation conflicts with the implemented healthcheck and names no mechanism

**Evidence.** Plan line 136 requires "host validation for `brightflag-mcp.tqaentry.com`". The server
has no host allow-list — no `AllowedHosts`, `HostFiltering`, or equivalent anywhere in `src/`. The
implemented healthcheck sends `Host: localhost`
(`BrightFlagProxyMCPServer/deploy/local/compose.yaml:71`), and Caddy reaches the backend by
container alias `brightflag-mcp-server:8080`. An allow-list containing only the canonical hostname
breaks the healthcheck and any container-name access.

**Correction.** Name the configuration key, require the allow-list to include the loopback or health
host and the container alias — as OntologyService's `ALLOWED_HOSTS` does
(`LocalAI/docs/setup-ontology-mcp-windows.ps1:60,94`) — and add a matching test to Prompt 19's proof
list. Note that Caddy's site block already matches on host, so the server-side check covers only
direct container access.

**Blocks implementation:** no.

### M2 — LocalAI-generated Compose silently drops the implemented container hardening and limits

**Evidence.** Plan line 181 ("Generate the Compose project under `C:\mcp\brightflag`") and the
responsibility list at lines 165–194 never mention `user`, `read_only`, `cap_drop`,
`no-new-privileges`, `tmpfs`, CPU and memory ceilings, log rotation, or the healthcheck. All are
present at `BrightFlagProxyMCPServer/deploy/local/compose.yaml:10-16, 66-88`, and none is in the
supersession table.

**Why it matters.** The plan says the later stages "override only the named home-lab decisions"
(line 154). Dropping unnamed behaviour is an unrecorded regression, not an override.

**Correction.** Require the generated Compose to carry the same service hardening, resource
ceilings, logging, and healthcheck as `deploy/local/compose.yaml`, or add explicit supersession rows
for each item deliberately dropped.

**Blocks implementation:** no.

### M3 — LAN-readable LocalStack exposes the fixed token and cursor key; consequence unstated

**Evidence.** Plan lines 36–39 place the fixed token, cursor-signing key, and BrightFlag
integration-test token in LocalStack, and accept that LocalStack stays LAN-reachable and
unauthenticated (`LocalAI/docs/setup-mcp-host-windows.ps1:35-37` — "LAN only ... it is
unauthenticated"; any credentials accepted, `:583-584`). The table row at plan line 119 supersedes
Prompt 17's isolation requirement, which is settled.

What is missing is in the consequence paragraph at lines 98–100, which commits to stating
fixed-token consequences plainly. It lists shared identity, audit attribution, rate-limit bucket,
and plan scope. It does not state that any LAN host can read the fixed token directly from
LocalStack — making the fixed-token check equivalent to no authentication for a LAN-resident caller
— or that the cursor-signing key is equally readable, which voids the cursor-integrity property the
signing key exists to provide.

**Correction.** Add both statements to the Prompt 18 consequence paragraph and to the LocalStack
acceptance row. No mitigation is requested.

**Blocks implementation:** no.

### M4 — The builder narrative process is misstated

**Evidence.** Plan line 156: "Add a new narrative fragment for the accepted design when
implementation begins." CLAUDE.md §2 states the fragment is produced by the Project Narrative action
from the `narrative-required` label plus the three pull-request body headings, and that hand-writing
a fragment is "Only needed when an entry was missed, or when correcting the record."

**Why it matters.** A hand-written fragment plus the label produces a duplicate entry; the label
without the three headings fails the workflow visibly. Neither is recoverable after merge.

**Correction.** Replace line 156 with: apply `narrative-required` before merge and complete `##
Narrative Context`, `## Narrative Decision`, and `## Narrative Consequences` in the pull-request
body via the template; the fragment arrives on a follow-up draft pull request. Keep the hand-written
fragment as the missed-entry fallback. Plan lines 157–158 are correct as written; note explicitly
that they scope to the builder repository only — LocalAI's `Narrative.md` is hand-edited and
reverse-chronological (`LocalAI/CLAUDE.md`, "Narrative.md" section), so plan line 53 and plan line
157 describe two different mechanisms.

**Blocks implementation:** no.

### M5 — The `plans/README.md` open-decisions list is not covered

**Evidence.** Plan line 153 says only "Add Stages 18 and 19 to the builder README and
`plans/README.md`". CLAUDE.md requires "the table plus open-decisions list in `plans/README.md` in
the same change". That list is prose introduced by a hard-coded count — "Nine points where the
prompt sequence leaves a choice" in [plans/README.md](../plans/README.md) — and Stages 18 and 19
surface at least three new ones: fixed-token exclusivity, the shared-realm caller contract, and
deployment ownership moving to LocalAI.

**Correction.** Name the open-decisions list and its count in Workstream 4, and add the new entries.

**Blocks implementation:** no.

### M6 — The AWS CLI dependency on the Windows host is unstated

**Evidence.** Plan items at lines 172–179 require retrieving and materialising three LocalStack
secrets. Prompt 17's implementation shells out to the AWS CLI
(`BrightFlagProxyMCPServer/deploy/local/scripts/Materialize-Secrets.ps1:34-36`). Neither
`LocalAI/docs/setup-mcp-host-windows.ps1`'s prerequisite list (`:53-63`) nor the plan installs or
checks for it, and the cited model script has no secret handling at all.

**Correction.** Add an explicit prerequisite and preflight check for the AWS CLI, or name the
alternative — a direct `Invoke-RestMethod` against the Secrets Manager JSON API — and attach the
never-print-values and redact-errors rule to whichever is chosen.

**Blocks implementation:** no.

### M7 — Workstream 1 item 3 is stale

**Evidence.** Plan lines 68–69 instruct incorporating "the two newer LocalAI `origin/main` commits
containing the Keycloak proposal and private Git build authentication". LocalAI is on `main` at
`5209120` with a clean tree; `19ba37a` (Keycloak documentation) and `21f3eec` (BuildKit
`GIT_AUTH_TOKEN`) are both already ancestors.

**Correction.** Replace with the recorded baseline commit `5209120`, per plan item 5 (line 71).

**Blocks implementation:** no.

---

## Low and observation

- **L1 — Rollback retention has no named store.** Plan lines 191–193 record a "previous deployable
  image" but name no file and no protection against `docker image prune`, which would silently break
  rollback. Name the record file and state the pruning consequence.
- **L2 — Building from a full commit SHA via BuildKit's Git context is the load-bearing step and has
  never been exercised here.** The cited model only ever builds a branch. Add it as the first manual
  gate rather than an assumption.
- **L3 — `reviews/` is a new top-level directory** not described in CLAUDE.md §1, `START-HERE.md`,
  or `README.md`. Observation only.
- **L4 — Caddy validation touches a container the script does not own.** Plan item 17 (line 186)
  requires validating the complete Caddy configuration, which means `docker compose -f
  C:\mcp-host\docker-compose.yml exec -T caddy caddy validate` followed by a reload. That is
  compatible with non-ownership but should say so explicitly — read-only exec plus reload, never
  restart — as Prompt 17 lines 122–123 did.
- **L5 — "Payment always enabled" has no named mechanism.** Prompt 17 disables payment by leaving
  `MarkInvoicePaidRoles` unset (`BrightFlagProxyMCPServer/deploy/local/compose.yaml:54` sets only
  the read grant). Naming `BrightFlag__Authorization__MarkInvoicePaidRoles__0` in Prompt 19 would
  make plan line 118 verifiable rather than declarative; the manual gates at plan lines 268–269
  otherwise carry the whole claim.

---

## Areas with no material findings

- **Repository boundaries.** The changeable and unchangeable file lists (plan lines 46–62) are
  correct and internally consistent, and no workstream step authorises a direct edit to
  `BrightFlagProxyMCPServer`, `OntologyService`, or `OntologyServerBuilder`. The one pressure point
  is B2 and H1, where a contradictory active artifact and an unaddressed shared-service change make
  such an edit the path of least resistance; the fix is to name the change in the prompt, not to
  relax the boundary.
- **Exclusivity as stated.** Plan lines 81–84 and 31 state the no-fallback requirement precisely and
  without a precedence loophole. The defect is in what the plan omits about the second code path
  (H3), not in the requirement.
- **Stage sequencing and structure.** Prompt 17 is genuinely merged in both the builder (`b6e7d19`)
  and the server (`b6673a4`), so the Stage 18 and 19 precondition holds. Adding a prompt and plan
  pair plus index rows is the correct shape, and the prescribed plan structure (plan line 102)
  matches CLAUDE.md §1.
- **Prose conventions.** No emoji, absolute dates, and wrapping within 100 columns throughout.
- **Deferred issues #45, #46, and #47.** Deferring them is legitimate. Two places where the plan
  contradicts its own deferral are reported above — H2 for #45 and H4 for #46. #47 is consistent.

[plan]: 2026-08-02-ai-mcp-server-development-deployment-plan.md
