# Adversarial review plan — `ai-mcp-server` development deployment

Status: proposed; revised after adversarial review; not yet approved for implementation.

Date: 2026-08-02

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
- The canonical endpoint is `http://brightflag-mcp.tqaentry.com/mcp`.
- Caddy permits private source addresses and rejects non-private sources for the BrightFlag route.
- The BrightFlag MCP endpoint uses an explicit plaintext transport mode on the LAN. It is not
  represented as an HTTPS trusted-proxy deployment.
- Keycloak remains HTTPS at `https://auth.tqaentry.com/realms/homelab`.
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
  and records both values, following the approach in
  `LocalAI/docs/setup-ontology-mcp-windows.ps1`; it does not require a published GHCR digest.
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
3. Incorporate the two newer LocalAI `origin/main` commits containing the Keycloak proposal and
   private Git build authentication before building on those files.
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
- a deterministic shared home-lab caller identity;
- read and payment capabilities for that fixed identity;
- a simple bearer challenge suitable for clients configured with a fixed header, without Keycloak
  discovery or OAuth protected-resource metadata in fixed-token mode;
- retained Keycloak issuer, audience, signature, lifetime, role, and scope validation;
- both `brightflag.read` and `brightflag.payment` for the designated Keycloak development user;
- rejection of arbitrary `homelab` realm tokens that are not addressed to `brightflag-mcp` or do
  not carry the required roles;
- no environment or profile restriction on selecting fixed-token mode; and
- no automatic hardening or production-suitability check for this explicitly selected shortcut.

The prompt must state the fixed-token consequences plainly. Every token holder shares one caller
identity, audit attribution, rate-limit bucket, and caller-bound plan scope. This is accepted for
the physically controlled home-lab development network.

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

Prompt 19 requires:

- `http://brightflag-mcp.tqaentry.com/mcp` as the canonical endpoint;
- an explicit `http://` Caddy site address so Caddy does not introduce automatic HTTPS;
- an explicit server plaintext transport mode that accepts the HTTP external resource URI without
  claiming an HTTPS trusted-proxy topology;
- no application profile guard that rejects that explicitly selected plaintext mode;
- a private-source matcher that rejects Internet clients;
- a BrightFlag container on external network `mcp-public` with no published backend port;
- Streamable HTTP proxy settings that prevent buffering;
- host validation for `brightflag-mcp.tqaentry.com`;
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

1. Add Stages 18 and 19 to the builder README and `plans/README.md`.
2. State that both run after Stage 17 and override only the named home-lab decisions.
3. Keep existing prompt text and historical narrative entries unchanged.
4. Add a new narrative fragment for the accepted design when implementation begins.
5. Compile `Narrative.md` from fragments; never edit it directly.
6. Validate prompt/plan pairing, stage boundaries, indexes, and 100-column wrapping.

## Workstream 5 — Add the LocalAI deployment owner

Add `LocalAI/docs/setup-brightflag-mcp-windows.ps1`, modelled on
`setup-ontology-mcp-windows.ps1`. Update LocalAI documentation and narrative in the same change.

The script will:

1. Accept repository URL, Git ref, work directory, auth mode, Keycloak user, canonical hostname,
   and LocalStack secret identifiers.
2. Resolve the supplied Git ref once to a full 40-character commit using transient repository
   authentication.
3. Build that exact commit with a transient BuildKit `GIT_AUTH_TOKEN` secret and a commit-derived
   image tag.
4. Never persist that GitHub credential in LocalStack, Compose, generated files, or logs.
5. Generate and store a cryptographic fixed MCP token when absent.
6. Generate and store a cryptographic cursor-signing key when absent.
7. Refuse deployment until the BrightFlag integration-test token has been seeded.
8. Reuse all three values until explicit rotation is requested.
9. Retrieve secrets without printing their values.
10. Atomically materialise required runtime files under `C:\mcp\brightflag\secrets`.
11. Apply and verify restrictive NTFS ACLs.
12. Generate the Compose project under `C:\mcp\brightflag`.
13. Configure exactly one authentication mode without an environment/profile restriction.
14. Enable both read and payment capabilities in either mode.
15. Attach only to `mcp-public` and expose only an internal container port.
16. Own only the BrightFlag Caddy fragment under `C:\mcp-host\caddy\conf.d`.
17. Validate the complete Caddy configuration before reloading it.
18. Preserve Caddy, Keycloak, PostgreSQL, LocalStack, OntologyService, all unrelated fragments,
    Jamie's SSH access, and unrelated containers.
19. When the requested ref resolves to a new commit, build its commit-tagged image and recreate
    only BrightFlag.
20. Record the requested ref, resolved commit, image tag, image ID, and previous deployable image.
21. Roll back by recreating only BrightFlag from the retained previous image, without rebuilding a
    moving ref.
22. Support idempotent reruns, explicit rotation, diagnostics, and BrightFlag-only stop/removal.

## Workstream 6 — Configure Keycloak idempotently

The LocalAI deployment path ensures:

- realm `homelab`;
- resource client `brightflag-mcp`;
- the existing public `mcp-client`, when compatible;
- device authorization enabled;
- an exact `brightflag-mcp` audience mapper;
- roles `brightflag.read` and `brightflag.payment`;
- both roles assigned to the named development user;
- no public-client secret; and
- no password, access token, or refresh token in generated configuration.

The caller obtains a token through device flow or its helper and supplies the token to the MCP
client as an `Authorization: Bearer` header. The plaintext MCP endpoint does not weaken the HTTPS
Keycloak issuer, discovery, token, or JWKS endpoints.

## Review disposition

The 2026-08-02 adversarial review produced these changes to the plan:

- use an explicit plaintext application transport mode instead of describing HTTP as an HTTPS
  trusted-proxy deployment; and
- resolve a requested Git ref before building, use a commit-derived image identity, retain the
  previous image, and make rollback independent of a moving ref.

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

The initial work still configures the Caddy matcher, the shared-realm Keycloak option, and the
minimum LocalStack secret lifecycle described above. End-to-end source-address proof, further
Keycloak client separation, and secret-lifecycle hardening are deferred to those issues.

## Verification plan

### Repository checks

- prompt and plan pair one-to-one;
- plan sections follow the binding builder structure;
- Markdown wraps at 100 columns;
- narrative compilation and validation pass;
- PowerShell parses successfully;
- generated Compose contains no unresolved placeholders or secret values;
- the model contains one BrightFlag container, no published backend port, and `mcp-public`;
- auth modes are mutually exclusive;
- fixed-token mode has no environment/profile restriction;
- the explicit plaintext mode accepts the HTTP external resource URI without claiming HTTPS;
- the build uses the recorded full commit rather than re-reading a moving ref; and
- no new diffs appear in the three read-only repositories relative to the recorded baselines.

### Manual gates on `ai-mcp-server`

- a selected under-development branch builds and its resolved revision is recorded;
- LocalStack remains usable from permitted LAN machines;
- the three secrets exist, materialise, and are consumed without being embedded in Compose;
- fixed-token mode grants read and payment access;
- Keycloak device flow grants read and payment access;
- neither mode falls back to the other;
- the canonical plaintext endpoint works from a LAN client;
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
