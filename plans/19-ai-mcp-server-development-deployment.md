# Stage 19 — `ai-mcp-server` development deployment

Source: `BrightFlagProxyMCPBuilder/prompts/19-ai-mcp-server-development-deployment.md`

## Context

Stage 17 gave the server repository its own local deployment: a PowerShell entry point under
`deploy/local` that pulled an immutable GHCR digest, fronted by shared Caddy over HTTPS, with
payment disabled and real-secret bootstrap blocked until LocalStack was proved unreachable from the
LAN. That deployment was written for a host whose shared infrastructure has since moved on, and it
duplicates work the LocalAI repository already does for every other MCP service on the same box.

Stage 19 moves deployment ownership out of this server entirely. One LocalAI script builds a named
commit of the private repository, materialises three LocalStack secrets, generates the Compose
project and one Caddy fragment, and can roll back to a retained image. The server keeps only what it
must: an honest transport posture for a plaintext LAN endpoint, host validation, both capabilities
in both authentication modes, and documentation that says which deployment is real.

The posture this stage adopts is deliberately open. The value of the plan is that each opening is
named where a reader meets it, rather than being discovered later by someone who assumed otherwise.

## Preconditions

- Stages 1–11 and 17 are merged, and Stage 18's exclusive two-mode authentication is merged and
  green.
- The shared host provides Docker Desktop in Linux-container mode, the external `mcp-public`
  network, the shared Caddy at `C:\mcp-host` importing `conf.d`, Keycloak 26 at
  `https://auth.tqaentry.com`, and LocalStack Secrets Manager.
- The Keycloak `homelab` realm, the `brightflag-mcp` resource client, the `tid` and flat-`roles`
  mappers, and a development user holding both roles exist or are created by the deployment.
- The BrightFlag integration-test service token has been seeded into LocalStack; no production
  BrightFlag URL or credential exists for this deployment.
- The DNS name `brightflag-mcp.tqaentry.com` resolves to the shared Caddy for intended clients.
- `ai-mcp-server` runs PowerShell 7.6.4. The LocalAI deployment script targets 7.6 or later and
  declares that minimum; Windows PowerShell 5.1 is not a target.
- Issues [#45][issue-45], [#46][issue-46], and [#47][issue-47] are open and non-blocking.

## Scope in

Deleting the `deploy/local` deployment artifacts and `docs/deploy-local.md`; an explicit plaintext
HTTP transport mode with no profile guard; host validation for exactly three names; documentation of
the LocalAI-owned deployment contract, its generated Compose and Caddy fragment, its resolved-commit
build and retained-image rollback, and its three LocalStack secret identifiers; payment enabled by a
named role value in both authentication modes; explicit recording of the omitted hardening, the
unproven private-source matcher, the LAN-readable LocalStack, and the plaintext credentials.

## Scope explicitly out

Editing `LocalAI` from this repository, or copying its script here; reintroducing any deployment
entry point in the server repository; removing GitHub-hosted CI or GHCR publication; changing
LocalStack networking, ports, or firewall rules; owning or restarting Caddy, Keycloak, PostgreSQL,
or LocalStack; proving the private-source matcher, which belongs to [#45][issue-45]; dedicated
Keycloak client separation, which belongs to [#46][issue-46]; secret-lifecycle hardening, which
belongs to [#47][issue-47]; any production BrightFlag configuration; contingent Stages 12 or 13.

## Work items

### 1. Retire the server-owned deployment

Delete `deploy/local/compose.yaml`, the `deploy/local` Caddy template,
`deploy/local/manual-gates.md`, every script under `deploy/local/scripts/`, and
`docs/deploy-local.md`. Remove or rewrite every
reference that presented them as current. Mark anything retained for history as superseded and
inert, and describe GHCR publication as CI evidence rather than as a deployment path.

### 2. Add the explicit plaintext transport mode

Extend the transport posture enumeration with a third explicitly selected value beside direct TLS
and trusted proxy. It accepts an `http://` external resource URI, asserts no TLS termination and no
fronting-proxy trust, and is refused as a default or fallback. No deployment-profile validation may
reject it.

### 3. Add host validation

Accept exactly `brightflag-mcp.tqaentry.com`, `localhost`, and the healthcheck's container alias
from configuration; refuse every other `Host`. Keep the accepted set a configured list so the
deployment supplies it, and assert the configured healthcheck's `Host` value is inside it.

### 4. Document the LocalAI-owned deployment contract

Describe what the LocalAI script produces and what the server requires from it: the `mcp-public`
network and unpublished port, the `auth.tqaentry.com:host-gateway` mapping, the explicit `http://`
Caddy site, the non-buffering proxy settings, the three materialised secret files, and the named
read and payment role values. Name the script as the sole deployment owner.

### 5. Enable payment in both modes

Configure `BrightFlag__Authorization__MarkInvoicePaidRoles__0` alongside
`BrightFlag__Authorization__ReadApprovedInvoicesRoles__0`, so the payment grant is a named value
rather than the absence of an overlay. Prove the complete surface is reachable under both the fixed
token and a Keycloak token.

### 6. Record the open posture in the places a reader will meet it

The omitted container hardening and resource limits; the private-source matcher's unproven
effectiveness and its issue; the unauthenticated LAN-reachable LocalStack holding all three secrets;
the plaintext bearer credentials on the LAN; and the fact that a resolved commit makes a build
reproducible without making a branch immutable. State each once, plainly, where it is relevant.

## Tests

Map one-to-one to the prompt's Prove list:

1. assert no deployment entry point, Compose file, Caddy template, manual-gates file, or
   `deploy-local` document exists; grep the repository for references to them and assert every
   survivor is marked superseded and inert and that the LocalAI script is named as the owner;
2. select the plaintext transport mode, assert an `http://` resource URI is accepted with no HTTPS
   or trusted-proxy assertion, assert an unset posture still fails startup, and assert no profile
   value refuses the mode;
3. drive `Host` values through the transport: accept the canonical hostname, `localhost`, and the
   healthcheck alias; refuse others; and assert the configured healthcheck's `Host` is in the
   accepted set;
4. parse the generated Compose and assert one BrightFlag container, external `mcp-public`, no
   published port, the `auth.tqaentry.com:host-gateway` mapping, and the expected healthcheck;
5. assert the generated Compose has no unresolved placeholder and no secret value, and assert the
   absence of `user`, `read_only`, `cap_drop`, `security_opt`, `tmpfs`, CPU and memory limits, and
   log rotation, each with a test naming the omission as intentional;
6. assert the generated Caddy fragment's site address begins `http://`, carries the private-source
   matcher and non-buffering settings, changes no unrelated fragment or the root Caddyfile, and
   describes the matcher as unproven with its issue reference;
7. assert the deployment state record carries the requested ref, a full 40-character commit, the
   image tag, the image ID, and the previous image; build from a full commit and assert the image
   label reports it; and exercise rollback from the retained image with no ref resolution or
   rebuild;
8. materialise the three configured secret identifiers, assert atomic ACL-protected files consumed
   by the `File` sources, scan logs, errors, Compose, process arguments, and Docker metadata for the
   values, and assert no AWS SDK, LocalStack endpoint, or vendor secret provider is reachable from
   application code;
9. exercise the complete surface under both authentication modes, assert the payment grant comes
   from the named `MarkInvoicePaidRoles__0` value, and assert neither mode falls back to the other;
10. decode a Keycloak device-flow token and assert flat `tid` and `roles`, the `brightflag-mcp`
    audience, and the HTTPS issuer; and retrieve discovery and JWKS from inside the container
    through `auth.tqaentry.com`; and
11. grep configuration, fixtures, documents, and generated files for any BrightFlag production URL
    or credential and assert none; and assert this stage changes no LocalStack network, port, or
    firewall configuration.

Tests 1–3, 5, 9, and 11 are automatable in the server repository. Tests 4, 6, 7, 8, and 10 depend on
the LocalAI script's output or on the host: automate what can be asserted against generated files,
and label the rest as manual gates.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
test ! -e deploy/local && test ! -e docs/deploy-local.md && rg -n "deploy.local" docs/ README.md
```

The third command must find no path that is presented as current; any remaining hit must read as
superseded. Manual gates on `ai-mcp-server`, each recorded with its exact command and dated result:
a selected branch builds and records its resolved revision; a selected full commit builds and its
image label reports that commit; the three secrets materialise and are consumed without appearing in
Compose; fixed-token mode grants read and payment; a device-flow token grants read and payment and
carries only `iss`, `aud`, `tid`, and flat `roles`; discovery and JWKS succeed from inside the
container; neither mode falls back; the canonical plaintext endpoint works from a LAN client; the
healthcheck succeeds; the backend is unreachable directly; Streamable HTTP responses are not
buffered; advancing a branch replaces only BrightFlag and retains the previous image; rollback
restores it without rebuilding; host and Docker Desktop restart recovery works; and stop or removal
preserves every shared service. External proof that Caddy rejects a genuinely non-private source
belongs to [#45][issue-45] and is not an acceptance gate here.

## Stage boundary

Commit locally. Suggested message: `Adopt the LocalAI-owned development deployment`.

Use `narrative-required` when published. Do not push unless requested. Do not edit the LocalAI
repository from this stage, reintroduce a deployment entry point here, change LocalStack's exposure,
claim the private-source matcher works, configure a BrightFlag production endpoint, or begin
contingent Stages 12 or 13.

## Risks

- Deleting `deploy/local` while a reference to it survives leaves a reader with two deployments and
  one of them broken. The grep in the acceptance checks exists for exactly that.
- A plaintext transport mode is one refactor away from becoming a silent default for an unset
  posture. Keep the unset case failing and test it alongside the new mode.
- The private-source matcher looks like an Internet-exposure control and is not proven to be one.
  A public DNS name with an explicit `http://` site could become Internet-reachable if router
  forwarding changes; that risk is real, is tracked by [#45][issue-45], and must not be written up
  as closed.
- LocalStack holds the fixed token, the cursor-signing key, and the BrightFlag integration-test
  credential unauthenticated on the LAN. The practical boundary for all three is physical control of
  the network.
- Recording a resolved commit is not immutability. A branch build repeated later resolves a
  different commit, which is why rollback uses the retained image and never the ref.
- Payment is enabled against BrightFlag's integration-test environment. The one control that keeps
  this from touching real money is that no production URL or credential exists for this deployment;
  introducing one would silently change what this stage means.

[issue-45]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/45
[issue-46]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/46
[issue-47]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/47
