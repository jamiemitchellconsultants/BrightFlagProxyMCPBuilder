# Start here: set up your repository and coding agent

This guide is for learners who have not built an MCP server or connected a coding agent to a
repository before. The server you build is limited to reporting a BrightFlag schema for an ontology
service, retrieving invoices approved for payment, and marking one approved invoice as paid. When
you finish, you will have:

- a GitHub repository named `MyBrightFlagProxyMCPServer`;
- an optional local clone of that repository;
- one coding agent connected to the correct workspace;
- the supported .NET SDK available;
- a clear distinction between the builder repository and the server you will build; and
- a safe read-only task proving the setup works.

You do not need BrightFlag credentials. The learning sequence begins by acquiring the public
BrightFlag OpenAPI document as a reviewed, checked-in snapshot. It then uses synthetic invoices and
batches and a local fake endpoint implementing only the five BrightFlag operations the server is
allowed to call.

## 1. Create `MyBrightFlagProxyMCPServer` on GitHub

You need a [GitHub account](https://github.com/signup).

1. Sign in to GitHub.
2. Open [Create a new repository](https://github.com/new).
3. Choose your personal account or organisation as the owner.
4. Enter `MyBrightFlagProxyMCPServer` as the repository name.
5. Add a description such as `My guided BrightFlag proxy MCP server`.
6. Choose **Private** while learning unless you deliberately want it public.
7. Select **Add a README file**.
8. Leave `.gitignore` and licence unselected; Prompt 1 adds them deliberately.
9. Select **Create repository**.

`BrightFlagProxyMCPBuilder` contains instructions. `MyBrightFlagProxyMCPServer` is the separate
repository where the coding agent implements the service. Do not ask the agent to build the server
inside this builder repository.

## 2. Choose local or cloud work

| | Local agent | Cloud agent |
|---|---|---|
| Where it runs | On your computer | In a provider-hosted environment |
| How it receives code | You select a cloned folder | You authorise a GitHub repository |
| Sees uncommitted files | Yes | No |
| Can reach a local fake BrightFlag server | Yes | Only with extra hosted setup |
| How changes appear | In the local working tree | Usually on a remote branch or pull request |

A local agent is recommended for the first pass because the fake BrightFlag server, tests, Git
diff, and MCP stdio process are easier to inspect together. A cloud agent can still complete the
sequence, but it must keep integration tests self-contained inside its environment.

## 3. Clone the repository locally

### Recommended: GitHub Desktop

1. Install [GitHub Desktop](https://desktop.github.com/download/) and sign in.
2. Open `MyBrightFlagProxyMCPServer` on GitHub.
3. Select **Code**, then **Open with GitHub Desktop**.
4. Choose a local folder and select **Clone**.
5. Use **Repository → Show in Finder** on macOS or **Show in Explorer** on Windows.
6. Confirm the folder contains `README.md`.

### Command-line alternative

```bash
git clone https://github.com/YOUR-GITHUB-NAME/MyBrightFlagProxyMCPServer.git
```

Replace `YOUR-GITHUB-NAME` with the repository owner.

## 4. Install the development prerequisites

Install:

- the latest supported patch of the [.NET 10 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/10.0);
- [Node.js](https://nodejs.org/en/download) 20 or newer for the Project Narrative installer and
  validator;
- Git; and
- optionally Docker Desktop or another OCI-compatible container engine for Prompt 9.

Verify the local tools:

```bash
dotnet --info
```

```bash
node --version
```

```bash
git --version
```

Docker may be absent until the delivery stage. You do not need a database, a message broker, or any
BrightFlag SDK: every BrightFlag interaction in this sequence is plain HTTPS and JSON.

## 5. Connect one coding agent

Choose one route. Git and the checked-in files remain the source of truth if you switch later.

### Claude Code

1. Install [Claude Code](https://claude.com/download/) and sign in.
2. Open **Code**, choose **Local**, and select `MyBrightFlagProxyMCPServer`.
3. Keep normal permission prompts enabled.

### Codex on desktop

1. Install the current ChatGPT desktop application from
   [OpenAI's Codex getting-started page](https://openai.com/codex/get-started/).
2. Open Codex and add a project.
3. Select the locally cloned `MyBrightFlagProxyMCPServer` folder.
4. Choose a local environment and keep normal approval protections enabled.

### GitHub Copilot coding agent

1. Confirm your account and organisation permit the coding agent.
2. Open `MyBrightFlagProxyMCPServer` on GitHub.
3. Open the repository's **Agents** tab or
   [github.com/copilot/agents](https://github.com/copilot/agents).
4. Select `MyBrightFlagProxyMCPServer` and submit a small task.
5. Review the branch and pull request it creates.

For another coding agent, either select the cloned folder locally or authorise only the
`MyBrightFlagProxyMCPServer` repository for cloud work. Never paste BrightFlag bearer tokens,
GitHub tokens, client secrets, `.env` contents, or production invoice data into a prompt.

## 6. Prove the connection safely

Give the agent this read-only task:

```text
Read this repository and list the files you can see. Confirm the repository name, current branch,
and installed .NET SDK version. Do not create, edit, delete, commit, push, install, download, or
contact BrightFlag.
```

A correct result identifies `MyBrightFlagProxyMCPServer`, its `README.md`, and the .NET SDK. Stop
and fix the setup if the agent reports `BrightFlagProxyMCPBuilder`, cannot see the README, proposes
changing files, or tries to reach `app.brightflag.com`.

## 7. Begin the learning sequence

Keep this builder repository open so you can copy one prompt at a time.

1. Give the agent the complete [reusable contract](prompts/00-reusable-contract.md).
2. Wait for it to acknowledge the contract and restate the safety boundaries.
3. Give it [Prompt 1](prompts/01-solution-scaffold-and-invoice-contracts.md).
4. Review the diff and acceptance evidence.
5. Continue in the order listed in [README.md](README.md).

Do not paste every prompt at once. Each stage deliberately restricts scope and authority.

When a prompt says **commit locally but do not push**, a local agent may create a commit but must
not publish it. A cloud agent may require a remote branch; review that branch and its pull request
instead.

## 8. Introduce a BrightFlag tenant only when prompted

Prompt 1 acquires the public OpenAPI snapshot without credentials. Prompt 3 defines the runtime
origin and secret boundary. Until Prompt 3:

- make no BrightFlag request except Prompt 1's bounded administrative snapshot fetch;
- do not add a tenant hostname;
- do not request or store a bearer token;
- do not copy real invoices, vendors, matters, or batch exports into fixtures; and
- never call the payment-status endpoint against a live tenant.

BrightFlag credentials are issued by BrightFlag support at the request of your company's BrightFlag
administrator. When you later opt into a tenant, use a dedicated least-privilege integration
identity, a non-production tenant if one exists, and synthetic invoices.

Treat the payment-status endpoint as the sharpest edge in this project. It writes a payment
assertion into a legal-spend system that finance teams reconcile against real money, it accepts one
invoice per call, and it is not documented as idempotent. That is why the prompts require a plan,
an explicit confirmation, and at most one POST attempt for each atomically consumed plan.

## 9. Git words used by the guide

- **Repository:** the project and its version history.
- **Remote:** the shared repository hosted on GitHub.
- **Clone:** a local working copy.
- **Branch:** an isolated line of work.
- **Working tree:** files currently visible in a clone.
- **Commit:** a named snapshot in repository history.
- **Push:** publish local commits to a remote.
- **Pull request:** a review proposing that one branch be merged into another.
- **Diff:** the exact lines added, removed, or changed.

The coding agent can perform these actions, but you remain responsible for reviewing changes and
deciding what is published or merged.

## 10. Work safely

- Start with a private learning repository.
- Give the agent access only to the repository it needs.
- Keep approval and sandbox protections enabled.
- Never commit tokens, certificates, cookies, personal data, or production invoice payloads.
- Keep BrightFlag writes disabled until Prompt 6, and against a live tenant until you have a
  reviewed reason.
- Use branches for work that will be pushed.
- Read the diff before committing or merging.
- Require the checks named in every prompt.
- Do not approve a destructive command you do not understand.
- Do not merge merely because an agent created a pull request.

## 11. Common setup problems

### The agent sees the builder instead of the implementation repository

Select `MyBrightFlagProxyMCPServer`, not `BrightFlagProxyMCPBuilder` and not their parent folder.

### The .NET SDK is missing

Install the supported .NET 10 SDK and restart the terminal or desktop agent so it receives the
updated executable path.

### A cloud agent cannot reach the fake BrightFlag service

The fake server must run inside the same hosted task or test process. Do not expose a laptop port
or substitute a live BrightFlag endpoint.

### The agent wants to fetch the OpenAPI document on every run

Only the administrative snapshot command introduced in Prompt 1 may fetch
`/v3/api-docs/external`. Runtime startup reads the checked-in snapshot. If the agent adds a startup
fetch, reject the change.

### A local agent can edit but cannot push

Folder access and GitHub authentication are separate. Sign in through GitHub Desktop, GitHub CLI,
or the agent's supported integration. Do not send credentials through chat.

### A command works in Terminal but not in the desktop agent

Desktop apps may use a different environment. Check the agent's local environment and executable
path instead of repeatedly approving the same failing command.
