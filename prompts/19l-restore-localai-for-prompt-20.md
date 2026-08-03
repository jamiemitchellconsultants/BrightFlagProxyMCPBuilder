# Prompt 19L — Restore LocalAI for Prompt 20

Apply this prompt in the separately managed `LocalAI` repository after Prompt 19 has been applied
to `BrightFlagProxyMCPServer` and its Prompt 19 deployment changes have been played into LocalAI,
but before Prompt 20 is applied to the server.

The LocalAI repository already contains independently reviewed preparation for Prompt 20 and for
another MCP server. Prompt 19 deliberately rewound the BrightFlag-specific parts of that work to
the earlier plaintext, per-service-audience deployment. Restore only the BrightFlag deployment to
the LocalAI state Prompt 20 requires. Preserve every independent shared-host, Keycloak, Caddy,
LocalStack, ontology, other-MCP, and unrelated documentation change.

This is a LocalAI reconciliation stage, not a BrightFlag server stage. Do not edit
`BrightFlagProxyMCPServer` or this builder repository while playing it.

## Reconcile; do not revert

Do not restore an old tree, check out whole files from an earlier commit, revert a merge, or reset
LocalAI to a historical revision. Those operations would discard independent work that happened to
touch the same files.

Start from current LocalAI `main` on a new branch. Read LocalAI's binding agent instructions and its
current narrative before changing anything. Inventory the current diff and the relevant history,
then make the narrowest semantic edits necessary in:

- `docs/setup-brightflag-mcp-windows.ps1`;
- the BrightFlag and shared-audience portions of `README.md`;
- the BrightFlag portions of `docs/configure-keycloak.md`; and
- the new reverse-chronological entry required by LocalAI's narrative rules.

Do not assume those are the only files another person has changed. Preserve concurrent wording and
behaviour unless it conflicts directly with the contract below. If an overlap cannot be reconciled
without choosing between independent designs, stop and report the exact conflict instead of
choosing silently.

## Treat the shared host as an external prerequisite

The independently prepared LocalAI host deployment owns the shared identity and ingress substrate.
Verify, without rewriting it, that the current tree provides:

- realm `homelab` at issuer `https://auth.tqaentry.com/realms/homelab`;
- audience `homelab-mcp`, shared by the home-lab MCP servers;
- a realm-default client scope that puts `homelab-mcp` in `aud` and supplies constant flat `tid`
  and `roles` claims;
- realm-default capability scopes including `brightflag.read` and `brightflag.payment`;
- the public `mcp-client` fallback caller and dynamic-client-registration policy;
- the external Docker network `mcp-public`;
- Caddy's `auth.tqaentry.com` network alias on `mcp-public`; and
- shared Caddy support for one service-owned fragment under `C:\mcp-host\caddy\conf.d`.

This prompt does not own `docs/setup-mcp-host-windows.ps1`, the ontology deployment, another MCP
server, the root Caddyfile, the shared Keycloak stack, or LocalStack. If any prerequisite is absent,
stop with the missing property and the file that should own it. Do not manufacture a second realm,
audience, Caddy, or shared-infrastructure implementation inside the BrightFlag script.

## Restore the Prompt 20 BrightFlag deployment values

Keep `docs/setup-brightflag-mcp-windows.ps1` as the sole deployment owner for BrightFlag. It still
targets PowerShell 7.6 or later and still generates one BrightFlag container on external
`mcp-public` with no published host port.

Restore these Prompt 20 values:

| Setting or generated shape | Required value |
|---|---|
| Canonical MCP endpoint | `https://brightflag-mcp.tqaentry.com/mcp` |
| Keycloak issuer | `https://auth.tqaentry.com/realms/homelab` |
| Keycloak audience | `homelab-mcp` |
| `BrightFlag__CallerIdentity__Mode` in the Keycloak branch | `Keycloak` |
| Protected-resource URI | `https://brightflag-mcp.tqaentry.com/mcp` |
| Protected-resource read scope | `brightflag.read` |
| HTTP transport security | `TrustedProxy` |
| Trusted proxy name | an explicit declaration naming the Caddy ingress |
| Kestrel listener | internal `http://0.0.0.0:<container-port>` only |
| Caddy site address | bare `brightflag-mcp.tqaentry.com`, enabling automatic HTTPS |

The generated Caddy fragment contains the non-buffering Streamable HTTP setting
`flush_interval -1`. It contains no private-source matcher. The endpoint is intentionally public:
anyone who resolves the name can reach Caddy, and only the server's token validation authenticates
the caller. State that exposure where the endpoint is documented.

Remove the BrightFlag container's `auth.tqaentry.com:host-gateway` entry. Keycloak keeps one
canonical HTTPS issuer, and the container resolves it through the existing Caddy network alias on
`mcp-public`. Do not add a second Keycloak hostname or change the shared ingress to compensate.

Host validation remains unchanged. The deployment supplies exactly:

- `brightflag-mcp.tqaentry.com`;
- `localhost`; and
- the BrightFlag container alias used by the healthcheck.

The healthcheck sends a Host value inside that set.

## Stop the BrightFlag script from owning Keycloak

The BrightFlag script no longer creates a resource client, client roles, protocol mappers, user
assignments, realm defaults, or dynamic-registration policy. Remove its Keycloak administrator
credential path and every `kcadm` write.

Its Keycloak responsibility is a read-only preflight against the current realm's public discovery
document. A failed preflight must explain that authentication will fail; it must not mutate the
shared realm or silently create a per-service substitute.

The shared audience has a disclosed cost: `aud` no longer distinguishes BrightFlag from another
home-lab MCP server. The required tenant and role claims remain mandatory, and the read and payment
role configuration remains the authorization boundary. Do not add an authorization bypass or make
either claim optional.

## Retain Prompt 18's fixed-token mode

Prompt 20 does not remove or deprecate fixed-token authentication. The deployment still offers
exactly `fixed` and `keycloak`, selected once with no fallback in either direction. Default the
script to `keycloak` because the Prompt 20 deployment exists for conversational agents, but keep an
explicit `-AuthMode fixed` selection fully functional.

Do not restore any warning that the server lacks fixed-token support. Prompt 18 has now been applied
and that statement is false. Preserve the complete fixed-token environment contract, its mounted
file source, its configured subject and tenant, and both configured roles.

Both modes retain:

- `BrightFlag__Authorization__ReadApprovedInvoicesRoles__0`;
- `BrightFlag__Authorization__MarkInvoicePaidRoles__0`;
- the exact three allowed hosts;
- the explicitly supplied integration-test approved-invoice status with no default; and
- the complete four-tool, one-resource BrightFlag surface.

In Keycloak mode, the protected-resource metadata and challenge support an agent that performs
dynamic registration and authorization-code with PKCE. Device flow remains a fallback for tooling
that runs no OAuth flow. Fixed mode continues to use a caller-supplied fixed bearer header and does
not become an OAuth flow.

## Preserve the Prompt 19 ownership and operational controls

The following Prompt 19 work survives unchanged:

- resolve the configured private Git ref once to a full 40-character commit;
- build that exact commit and pass it as `SOURCE_REVISION`;
- record requested ref, resolved commit, image tag, image ID, and previous deployable image;
- retain the previous image and roll back BrightFlag without resolving or rebuilding;
- keep the GitHub credential transient and out of arguments, generated files, logs, LocalStack,
  Compose, and image metadata;
- retain the three configurable LocalStack identifiers for the fixed token, cursor key, and
  integration-test BrightFlag token without printing their values;
- atomically materialise ACL-protected files consumed by the server's `File` secret sources;
- keep LocalStack's unauthenticated LAN exposure recorded rather than claiming it is mitigated;
- keep the deliberately omitted container hardening and resource limits recorded; and
- keep BrightFlag integration-test only, with payment enabled only against that environment.

Do not replace the exact-commit build with GHCR deployment, add a host port, introduce an AWS SDK
into the server, harden the Compose opportunistically, or change LocalStack networking.

## Reconcile the documentation and narrative

Update only the relevant LocalAI documentation so it no longer presents the Prompt 19 plaintext
BrightFlag deployment as current:

- the BrightFlag endpoint is public HTTPS;
- BrightFlag uses the shared `homelab-mcp` audience;
- conversational agents use discovery, dynamic registration, and authorization-code with PKCE;
- device flow remains a non-agent fallback and fixed mode remains available;
- the private-source matcher and `host-gateway` entry are absent;
- Caddy authenticates nobody; the server's validation is the public endpoint's only authentication;
  and
- the shared audience, public reachability, LAN-readable LocalStack, and omitted hardening are each
  stated as accepted home-lab costs.

Preserve documentation for the other MCP server and the shared-host design. Do not replace whole
sections with historical text merely because an older BrightFlag revision contains desired wording.

Under LocalAI's narrative rules, prepend a new dated entry explaining that this is a selective
reconciliation after replaying Prompt 19, not a deletion of that decision. Keep the Prompt 19 entry
as historical evidence. Record which BrightFlag decisions Prompt 20 supersedes and which Prompt 19
ownership, fixed-token, build, rollback, secret, and open-posture decisions survive.

## Prove

- the changed-file inventory contains only the BrightFlag script, directly affected documentation,
  and the LocalAI narrative entry, with every unrelated independent change preserved;
- the PowerShell parser reports no syntax error under PowerShell 7.6 or later;
- the Keycloak branch emits `Mode=Keycloak`, audience `homelab-mcp`, the HTTPS resource URI,
  `TrustedProxy`, and an explicit Caddy proxy name;
- the fixed branch still emits every Prompt 18 fixed-token field and both role values, and no source
  or document claims that fixed-token support is absent;
- generated configuration still carries exactly three allowed hosts, the explicit approved status,
  both authorization roles, the active mode's file-backed secrets, no published port, and external
  `mcp-public`; the three configurable secret identifiers and any independently improved
  auth-independent materialisation lifecycle are not regressed;
- the generated Caddy site is the bare public hostname, contains `flush_interval -1`, and contains
  no private-source matcher;
- the generated Compose contains no `auth.tqaentry.com:host-gateway` entry and the current shared
  host script demonstrably supplies the Caddy network alias instead;
- the BrightFlag script performs no Keycloak administrative write and the read-only discovery
  preflight names the shared HTTPS realm;
- exact-commit build, source label, state, retained-image rollback, transient GitHub credential,
  LocalStack materialisation, ACL, and redaction logic remain reachable and unchanged in meaning;
- current BrightFlag documentation contains no plaintext canonical endpoint or per-service audience
  claim, while historical narrative entries remain intact; and
- no shared-host, ontology, other-MCP, LocalStack, root-Caddyfile, or unrelated LocalAI behaviour is
  changed by this reconciliation.

## Acceptance criteria

- LocalAI is again a valid deployment-side prerequisite for Prompt 20 without losing the independent
  Prompt 20 and other-MCP preparation already present in the repository.
- The BrightFlag script generates the public-HTTPS, shared-audience, trusted-proxy deployment Prompt
  20 expects, while fixed-token mode and every surviving Prompt 19 control remain functional.
- Documentation and narrative describe the current state and preserve the historical sequence.
- Every automated repository check and PowerShell syntax check passes. Windows, Docker Desktop,
  Caddy, Keycloak, LocalStack, DNS, certificate, and live agent checks not executed in this
  environment remain labelled manual gates with exact commands and expected results.

Commit locally. Follow LocalAI's own narrative and review rules. Do not push unless requested. Do
not deploy to the Windows host, mutate the live Keycloak realm, edit either server repository, or
begin Prompt 20's server changes from this LocalAI branch.
