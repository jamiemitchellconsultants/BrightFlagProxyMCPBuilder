# Adversarial review plan — `ai-mcp-server` development deployment

Status: approved and frozen for implementation. Any revision requires a new reviewed decision.

Date: 2026-08-02

Revised 2026-08-02, on operator instruction: the host's installed PowerShell version is recorded as
a settled requirement. This does not change any decision the review took; it removes an assumption
implementation had otherwise been left to make.

## Purpose

This plan adds later prompts to `BrightFlagProxyMCPBuilder` and direct host-deployment support to
`LocalAI`. It must not rewrite an existing prompt or edit `BrightFlagProxyMCPServer` directly.

The later prompts deliberately override conflicting home-lab work from Prompt 17 when played in
sequence. They do not prescribe the BrightFlag server's production deployment or authentication
strategy. The application will not enforce that distinction with a production-profile guard.

## Settled requirements

- `OntologyServerBuilder` is the builder for `OntologyService` and both are references only.
- `BrightFlagProxyMCPBuilder` owns prompts for changes to `BrightFlagProxyMCPServer`.
- `BrightFlagProxyMCPServer` is not edited directly during this work.
- `LocalAI` may be changed directly and owns the Windows host-deployment script.
- Existing prompts remain unchanged. Later prompts may overwrite their implemented work.
- Prompt 17 is merged and applied before Stages 18 and 19; the later stages replace conflicting
  implemented home-lab behavior rather than revising Prompt 17 history.
- The target is the Windows host `ai-mcp-server`, running Linux containers through Docker Desktop.
- `ai-mcp-server` runs PowerShell 7.6.4. Scripts written for this deployment may assume PowerShell
  7.6 or later and must declare it. Windows PowerShell 5.1 compatibility is not a requirement, and
  writing to the 5.1 subset in order to obtain it is a defect rather than caution, because it costs
  the 7.x-only constructs the scripts should use. A script whose control flow depends on inspecting
  `$LASTEXITCODE` after a native command must set `$PSNativeCommandUseErrorActionPreference`
  explicitly rather than inherit it: when that preference is true, a non-zero exit throws under
  `$ErrorActionPreference = "Stop"` before any `$LASTEXITCODE` check runs, and its default has
  moved between 7.x releases. It is `False` on 7.6.4, verified on that version.
- The canonical endpoint is `http://brightflag-mcp.tqaentry.com/mcp`.
- Caddy configures a private-source matcher for the BrightFlag route. Its effectiveness through
  Docker Desktop and router forwarding is unproven and tracked by issue [#45][issue-45].
- The BrightFlag MCP endpoint uses an explicit plaintext transport mode on the LAN. It is not
  represented as an HTTPS trusted-proxy deployment.
- Keycloak remains HTTPS at `https://auth.tqaentry.com/realms/homelab`.
- Keycloak is a functioning initial authentication option. It emits the server-required flat `tid`
  and `roles` claims, and the BrightFlag container resolves the HTTPS issuer through a self-owned
  host-gateway mapping.
- Authentication selects exactly one of fixed token or Keycloak, with no fallback.
- Both supported authentication modes permit the complete BrightFlag MCP surface, including
  payment planning and execution.
- Fixed-token mode is deliberately open for home-lab/dev/test use. It has no environment or
  production-profile restriction; selecting the mode is the operator's explicit decision.
- The fixed token, cursor-signing key, and BrightFlag integration-test service token are stored in
  LocalStack Secrets Manager.
- LocalStack remains reachable from other LAN machines. This deployment does not add an isolation
  gate or alter its firewall rules.
- The deployment resolves a configurable under-development Git ref once, builds that exact commit,
  and records both values. `LocalAI/docs/setup-ontology-mcp-windows.ps1` is a structural model
  only; it does not implement this immutable-build behavior.
- The BrightFlag endpoint and credentials can refer only to BrightFlag's integration-test
  environment. The developer is not issued a BrightFlag production API URL or production secret.

## Repository boundaries

### Files that may change

- numbered prompts, matching plans, indexes, and narrative artifacts in
  `BrightFlagProxyMCPBuilder`;
- `LocalAI/docs/setup-brightflag-mcp-windows.ps1` and necessary LocalAI documentation; and
- the LocalAI reverse-chronological `Narrative.md` entry required by that repository.

### Files that must not change

- every file in `BrightFlagProxyMCPServer`;
- every file in `OntologyService`; and
- every file in `OntologyServerBuilder`.

The implementation may inspect those repositories and may write prompts describing future changes
to `BrightFlagProxyMCPServer`.

## Workstream 1 — Prepare repository baselines

1. Inspect every working tree before editing and preserve unrelated work.
2. Confirm Prompt 17 is merged and use its implemented result as the server baseline.
3. Use LocalAI `main` baseline `5209120`, which already contains the Keycloak proposal and private
   Git build authentication commits.
4. Continue from the current BrightFlag builder branch unless its state has changed.
5. Record the exact baselines used by the review and implementation.

## Workstream 2 — Add Builder Prompt and Plan 18

Add:

- `prompts/18-home-lab-fixed-token-authentication.md`; and
- `plans/18-home-lab-fixed-token-authentication.md`.

Prompt 18 adds a fixed opaque bearer-token provider without removing Keycloak. It requires:

- exactly one configured authentication mode at startup;
- failure on an absent, unknown, ambiguous, or incomplete mode;
- no fallback when Keycloak is unavailable or rejects a token;
- a mounted token file for the fixed-token deployment path;
- constant-time token comparison and complete secret redaction;
- a deterministic shared home-lab caller identity from reviewed configuration: subject, configured
  tenant, and full read/payment roles;
- read and payment capabilities for that fixed identity;
- a simple bearer challenge suitable for clients configured with a fixed header, without Keycloak
  discovery or OAuth protected-resource metadata in fixed-token mode;
- retained Keycloak issuer, audience, signature, lifetime, role, and scope validation;
- both `brightflag.read` and `brightflag.payment` for the designated Keycloak development user;
- rejection of tokens whose audience lacks `brightflag-mcp` or whose flat `roles` claim lacks the
  required BrightFlag roles; with the shared `mcp-client`, roles are the service boundary because
  its audience mapper can also add `brightflag-mcp` to tokens for another MCP service;
- no environment or profile restriction on selecting fixed-token mode; and
- no automatic hardening or production-suitability check for this explicitly selected shortcut.

The prompt must state the fixed-token consequences plainly. Every token holder shares one caller
identity, audit attribution, rate-limit bucket, and caller-bound plan scope. Fixed-token mode does
not traverse the JWT `CallerTokenValidator`: it has no token expiry, revocation, or tenant-claim
corroboration, and establishes those values only from its reviewed static configuration. Any LAN
caller able to read the unauthenticated LocalStack secrets can obtain the fixed token and
cursor-signing key; this makes fixed-token authentication and cursor integrity a LAN trust
boundary. These consequences are accepted for the physically controlled home-lab development
network without additional mitigation.

The proof list must test that startup enables exactly one authentication branch, that the selected
fixed-token branch constructs that static caller identity, and that Keycloak tokens never reach it.

The matching plan follows the repository's required structure and maps tests one-to-one to the
prompt's proof list.

## Workstream 3 — Add Builder Prompt and Plan 19

Add:

- `prompts/19-ai-mcp-server-development-deployment.md`; and
- `plans/19-ai-mcp-server-development-deployment.md`.

Prompt 19 explicitly supersedes these Prompt 17 home-lab decisions:

| Prompt 17 | Prompt 19 |
|---|---|
| Immutable GHCR digest | Resolve a configurable Git ref and build its exact commit |
| Keycloak only | Exclusive fixed-token or Keycloak selection |
| Payment disabled | Read and payment always enabled |
| LocalStack LAN isolation required | Existing LAN-accessible LocalStack accepted |
| `edge_net` | `mcp-public` |
| `C:\mcp\edge` | `C:\mcp-host` |
| HTTPS MCP resource | Plaintext LAN MCP resource |
| HTTPS trusted-proxy transport | Explicit plaintext home-lab transport |
| Dedicated Keycloak realm | Shared `homelab` realm |
| Server `deploy/local` owns deployment | LocalAI script is the sole deployment owner |
| Container hardening and resource limits | Intentionally omitted from this open home-lab Compose |

Prompt 19 requires:

- `http://brightflag-mcp.tqaentry.com/mcp` as the canonical endpoint;
- an explicit `http://` Caddy site address so Caddy does not introduce automatic HTTPS;
- an explicit server plaintext transport mode that accepts the HTTP external resource URI without
  claiming an HTTPS trusted-proxy topology;
- no application profile guard that rejects that explicitly selected plaintext mode;
- a private-source matcher whose effectiveness through Docker Desktop is expressly unproven and
  tracked by issue [#45][issue-45], rather than claimed as a completed rejection property;
- a BrightFlag container on external network `mcp-public` with no published backend port;
- Streamable HTTP proxy settings that prevent buffering;
- host validation that permits `brightflag-mcp.tqaentry.com`, `localhost`, and the container alias
  used by the healthcheck;
- a healthcheck compatible with that host validation;
- a self-contained `extra_hosts` mapping of `auth.tqaentry.com` to `host-gateway`, so the container
  can discover and fetch HTTPS Keycloak JWKS without modifying the shared Caddy host deployment;
- full payment configuration, including a named `BrightFlag__Authorization__MarkInvoicePaidRoles__0`
  value alongside the corresponding read role;
- retirement by the future Prompt 19 application of `deploy/local/compose.yaml`, its Caddy
  template, `manual-gates.md`, every `deploy/local/scripts/` deployment script, and
  `docs/deploy-local.md`; the prompt replaces them with the LocalAI-owned deployment and marks any
  retained deployment reference as superseded and inert;
- deliberate omission of Prompt 17's container user, read-only root filesystem, capability drop,
  no-new-privileges, `tmpfs`, CPU/memory limits, and log rotation from this open home-lab Compose;
- a configurable private repository URL and Git ref;
- resolving the requested ref once to a full commit before the build;
- building and tagging the exact resolved commit, recording the requested ref, commit, and image
  identity, and retaining the prior image for rollback;
- three configurable LocalStack secret identifiers;
- atomic materialisation into ACL-protected runtime files;
- no AWS SDK or LocalStack dependency in application code;
- Keycloak issuer `https://auth.tqaentry.com/realms/homelab`;
- Keycloak audience/resource client `brightflag-mcp`;
- device authorization through the public caller client;
- read and payment roles for the designated development user;
- no production BrightFlag configuration or credential; and
- no LocalStack network, port, or firewall change.

## Workstream 4 — Update Builder sequencing and narrative

1. Add Stages 18 and 19 to the builder README and the table and open-decisions list in
   `plans/README.md`, updating its stated count.
2. State that both run after Stage 17 and override only the named home-lab decisions.
3. Keep existing prompt text and historical narrative entries unchanged.
4. For the Builder decision pull request, apply `narrative-required` before merge and complete the
   three `## Narrative …` sections from the pull-request template. Project Narrative creates the
   follow-up fragment pull request; hand-write a fragment only for a missed entry or correction.
5. For LocalAI, update its hand-maintained reverse-chronological `Narrative.md` under that
   repository's instructions.
6. Compile the Builder `Narrative.md` from fragments; never edit it directly.
7. Validate prompt/plan pairing, stage boundaries, indexes, open decisions, and 100-column wrapping.

## Workstream 5 — Add the LocalAI deployment owner

Add `LocalAI/docs/setup-brightflag-mcp-windows.ps1`, modelled on
`setup-ontology-mcp-windows.ps1`. Update LocalAI documentation and narrative in the same change.

The script will:

1. Accept repository URL, Git ref, work directory, auth mode, Keycloak user, canonical hostname,
   and LocalStack secret identifiers.
2. Require Git for Windows and the AWS CLI in preflight.
3. Resolve the supplied Git ref once to a full 40-character commit using `git ls-remote` with
   `GIT_ASKPASS`, a credential helper, or an equivalent mechanism that keeps the credential out of
   the repository URL and process arguments.
4. Build that exact commit with a transient BuildKit `GIT_AUTH_TOKEN` secret, a commit-derived
   image tag.
5. Pass the resolved commit as `SOURCE_REVISION` so the image label reports the built source.
6. Never persist the GitHub credential in LocalStack, Compose, generated files, logs, or command
   arguments.
7. Generate and store a cryptographic fixed MCP token when absent.
8. Generate and store a cryptographic cursor-signing key when absent.
9. Refuse deployment until the BrightFlag integration-test token has been seeded.
10. Reuse all three values until explicit rotation is requested.
11. Retrieve secrets without printing values and redact AWS CLI errors.
12. Atomically materialise required runtime files under `C:\mcp\brightflag\secrets`.
13. Apply and verify restrictive NTFS ACLs.
14. Generate the Compose project under `C:\mcp\brightflag`, including its healthcheck and the
    deliberate open-home-lab omissions named above.
15. Configure exactly one authentication mode without an environment/profile restriction.
16. Enable both read and payment capabilities in either mode.
17. Attach only to `mcp-public` and expose only an internal container port.
18. Own only the BrightFlag Caddy fragment under `C:\mcp-host\caddy\conf.d`.
19. Validate the complete Caddy configuration by read-only `caddy validate` execution in the shared
    Caddy container, then reload it without restarting the shared service.
20. Preserve Caddy, Keycloak, PostgreSQL, LocalStack, OntologyService, all unrelated fragments,
    Jamie's SSH access, and unrelated containers.
21. Preserve every existing mapper, scope, grant, and flow on the shared `mcp-client` other than
    the explicitly added BrightFlag audience and role-claim mappings.
22. When the requested ref resolves to a new commit, build its commit-tagged image and recreate
    only BrightFlag.
23. Record the requested ref, resolved commit, image tag, image ID, and previous deployable image
    in `C:\mcp\brightflag\deployment-state.json`.
24. Retain the recorded current and previous image tags; do not prune them automatically, and state
    that manual image pruning can remove rollback capability.
25. Roll back by recreating only BrightFlag from the retained previous image, without rebuilding a
    moving ref.
26. Support idempotent reruns, explicit rotation, diagnostics, and BrightFlag-only stop/removal.

## Workstream 6 — Configure Keycloak idempotently

The LocalAI deployment path ensures:

- realm `homelab`;
- resource client `brightflag-mcp`;
- the existing public `mcp-client`, when compatible;
- device authorization enabled;
- an exact `brightflag-mcp` audience mapper;
- roles `brightflag.read` and `brightflag.payment`;
- both roles assigned to the named development user;
- a hardcoded `tid` claim mapper containing the exact configured BrightFlag tenant value;
- a mapper that projects the `brightflag-mcp` client roles into a flat top-level `roles` array;
- no public-client secret; and
- no password, access token, or refresh token in generated configuration.

The caller obtains a token through device flow or its helper and supplies the token to the MCP
client as an `Authorization: Bearer` header. The plaintext MCP endpoint does not weaken the HTTPS
Keycloak issuer, discovery, token, or JWKS endpoints. The generated BrightFlag Compose uses
`extra_hosts: auth.tqaentry.com:host-gateway` so this HTTPS issuer remains reachable from inside the
container. With the shared caller client, its audience mapper is not a service-separation boundary;
the required BrightFlag roles are the boundary. Future dedicated-client separation and additional
hardening remain in issue [#46][issue-46].

## Review disposition

The 2026-08-02 adversarial review produced these changes to the plan:

- use an explicit plaintext application transport mode instead of describing HTTP as an HTTPS
  trusted-proxy deployment; and
- resolve a requested Git ref before building, use a commit-derived image identity, retain the
  previous image, and make rollback independent of a moving ref.

The subsequent review also requires functional Keycloak `tid` and flat-role mappers, self-contained
container-to-issuer resolution, explicit retirement of Prompt 17 deployment artifacts, and separate
Builder, LocalAI, and future-prompt verification classes. It records the shared-client audience
limitation, fixed-token/LocalStack consequences, source-revision label, AWS CLI prerequisite,
healthcheck host set, narrative workflow, and deployment-state record explicitly.

The following adversarial recommendations are declined. These are explicit home-lab/dev/test
decisions and require no further mitigation in this plan:

- fixed-token mode remains an unrestricted shortcut with full access and no production-profile
  guard;
- later prompts may override implemented Prompt 17 home-lab decisions;
- no Prompt 17 prompt text or other prompt history is rewritten;
- existing LocalStack reachability and the intentionally open container posture are accepted;
- plaintext bearer-token exposure on the physically controlled LAN is accepted; and
- pre-existing repository differences are not part of this plan's acceptance baseline.

## Deferred, non-blocking issues

These follow-ups provide visibility and do not block the home-lab deployment plan:

- [#45 — Verify LAN-only Caddy filtering for the public BrightFlag MCP hostname][issue-45];
- [#46 — Finalize the open Keycloak home-lab caller contract][issue-46]; and
- [#47 — Review LocalStack secret bootstrap and cursor-key rotation][issue-47].

The initial work configures the Caddy matcher, a functioning shared-realm Keycloak option, and the
minimum LocalStack secret lifecycle described above. End-to-end source-address proof, further
Keycloak client separation, and secret-lifecycle hardening are deferred to those issues.

## Verification plan

### Builder-repository checks

- prompt and plan pair one-to-one;
- plan sections follow the binding builder structure;
- Builder README and `plans/README.md` table and open-decisions list include both stages;
- Markdown wraps at 100 columns;
- narrative compilation and validation pass;
- the Builder decision pull request carries `narrative-required` and all three required narrative
  sections; and
- no new diffs appear in the three read-only repositories relative to the recorded baselines.

### LocalAI checks

- PowerShell parses successfully under 7.6 or later, and the script declares that minimum;
- generated Compose contains no unresolved placeholders or secret values;
- generated Compose declares one BrightFlag container, no published backend port, `mcp-public`, the
  expected healthcheck, the permitted host set, and the `auth.tqaentry.com:host-gateway` mapping;
- generated Compose deliberately omits the listed Prompt 17 container hardening and resource-limit
  settings;
- generated Caddy fragment uses the explicit HTTP site and private-source matcher without claiming
  that its effectiveness is proven;
- the deployment-state record contains the requested ref, full resolved commit, image tag, image
  ID, and previous image; and
- a build from a full commit passes `SOURCE_REVISION` and reports that commit in its image label.

### Future Prompt 18 and 19 implementation checks

These checks run only when their prompts are later applied to `BrightFlagProxyMCPServer`; they are
not claims about the Builder-only change:

- exactly one authentication branch is reachable after startup selection, with no fallback;
- fixed-token mode constructs the configured static caller identity and grants both capabilities;
- Keycloak validation accepts the configured `tid`, flat `roles`, issuer, and audience contract;
- the explicit plaintext mode accepts the HTTP resource URI without claiming HTTPS;
- the server deployment no longer has an active Prompt 17 `deploy/local` deployment owner; and
- full payment configuration names both read and mark-paid role values.

### Manual gates on `ai-mcp-server`

- a selected under-development branch builds and its resolved revision is recorded;
- a selected full commit builds through BuildKit and the resulting image label reports that commit;
- LocalStack remains usable from permitted LAN machines;
- the three secrets exist, materialise, and are consumed without being embedded in Compose;
- fixed-token mode grants read and payment access;
- a decoded device-flow token records only `iss`, `aud`, `tid`, and flat `roles`, and Keycloak
  device flow grants read and payment access;
- discovery and JWKS retrieval succeed from inside the BrightFlag container through
  `auth.tqaentry.com`;
- neither mode falls back to the other;
- the canonical plaintext endpoint works from a LAN client;
- the healthcheck succeeds with its permitted host value;
- the backend is unreachable directly;
- Streamable HTTP responses are not buffered;
- advancing and rebuilding a branch replaces only BrightFlag and retains its previous image;
- rollback restores the retained image without resolving or rebuilding the branch again;
- host and Docker Desktop restart recovery works; and
- stop/removal preserves every shared service and unrelated MCP deployment.

External proof that Caddy sees and rejects a genuinely non-private source is deferred to issue
[#45][issue-45] and is not a deployment acceptance gate.

## Adversarial review brief

The reviewer should not implement or edit this plan. Report findings with severity, evidence, and a
specific correction. In particular, challenge:

1. whether a later prompt can override each named Prompt 17 requirement without leaving mutually
   active artifacts or contradictory documentation;
2. whether fixed-token full access interacts safely and coherently with caller-bound payment plans;
3. whether the explicit absence of a fixed-token production-profile guard is implemented without
   accidentally reintroducing fallback or partial access;
4. whether Caddy can reliably distinguish private from Internet sources through Windows, Docker
   Desktop, router forwarding, and any hairpin path;
5. whether the public DNS name and explicit plaintext Caddy site can accidentally become
   Internet-reachable;
6. whether LocalStack secret bootstrap, retrieval, materialisation, rotation, and error output can
   leak values;
7. whether LAN-reachable unauthenticated LocalStack creates consequences the plan fails to state;
8. whether BuildKit Git authentication is transient in the actual Docker Desktop implementation;
9. whether the requested Git ref is resolved once and the exact commit is built, tagged, recorded,
   retained, and recoverable without a local clone or false immutability claim;
10. whether Keycloak device flow, audience mapping, role claims, and no-fallback behavior match the
    generated server's actual validation contract;
11. whether full payment access is genuinely present in both modes rather than merely documented;
12. whether setup, rerun, rotation, rollback, stop, and uninstall touch only BrightFlag-owned
    resources; and
13. whether every automated claim has an executable test and every host-dependent claim is labelled
    as a dated manual gate.

The review should also search for unstated assumptions, contradictions with binding repository
instructions, weakened acceptance checks, production side effects, and any action that would edit
`BrightFlagProxyMCPServer` directly.

[issue-45]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/45
[issue-46]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/46
[issue-47]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/47
