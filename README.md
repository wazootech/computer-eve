# Computer (Eve runtime)

<p align="center">
  <a href="https://docs.wazoo.dev">
    <img src="https://wazoo.dev/assets/wazoo.svg" alt="Wazoo Worlds" width="120" />
  </a>
  <br /><br />
  <em>Wazoo's operating computer for turning direction into shipped work.</em>
</p>

Computer is Wazoo's operational AI partner: the general assistant the whole team shares, modeled after the Enterprise computer from *Star Trek: The Next Generation*. It runs on [eve](https://eve.dev) and speaks through an authenticated web chat. Engineering is the one domain it runs as a supervised factory pipeline.

This repository, `wazootech/computer-eve`, is Computer's Eve runtime: the authenticated web chat and the GitHub factory, deployed on Vercel. Computer's persona, its prompt of record (`agent/instructions.md`), and its Discord channel live in the private [`wazootech/computer`](https://github.com/wazootech/computer) repository.

## Factory pipeline

Computer follows the official Eve software-factory shape:

```text
work item → classifier → researcher (when needed) → analyst → implementer → reviewer → draft PR
```

Each station has its own instructions, sandbox, tools, and structured output. The reviewer is independent, revision loops are capped, and Computer never merges or marks a pull request ready without a person. GitHub intake supports authorized mentions, deliberate candidate triage, human promotion, and CI-failure follow-up. The unattended pull-request summary comment is off by default: it ran on every branch anyone pushed, so an opened PR gets it only when `COMPUTER_PR_SUMMARY_AUTOREPLY` is `on`. Linear remains an optional channel.

## Risk-scaled review

The orchestrator uses `assess_review_depth` after analysis. Documentation-only changes take the light path, ordinary changes take standard review, and public API, security, permissions, migrations, data, deployment, runtime, or critical changes take deep review with targeted probes, repository checks, evidence fields, and an explicit human handoff. The reviewer can only upgrade the tier, never downgrade an actual high-risk diff.

## Deliberate intake

A normal work item does not start implementation automatically. An authorized maintainer applies `factory:candidate`; Computer classifies the issue with `set_intake_state` and moves it to `factory:queued`, `factory:needs-clarification`, `factory:duplicate`, or `factory:blocked`. A human then applies `factory:promoted` to a queued issue carrying `wayfinder:task`. Only that promotion starts the unattended pipeline, and it stops at a draft pull request.

Factory state transitions are durable, idempotent, and issue-scoped. The candidate session cannot create pull requests, close issues, mutate state through generic label tools, or write the shared factory brain.

## Durable run history

Every verified GitHub session emits a redacted lifecycle history under the private `run-history/v1/` Blob namespace. The stable run ID is the Eve session ID; `read_run_record` retrieves the immutable event set by that ID, and `list_run_history` lists repository-scoped summaries. Events use stable idempotency keys, so webhook retries do not duplicate records. The history records source delivery, turn and stage lifecycle, child-session lineage, approvals, and terminal outcome without raw issue content or credentials. Legacy Markdown records under `run-records/` remain a read-only compatibility fallback; new writes never update that legacy namespace.

## Memory and run records

Computer-specific curated memory, handoff artifacts, preferences, and redacted run records live in Vercel Blob, using reserved namespaces owned by dedicated tools. Run records contain stage events, approvals, outputs, token usage, and failures. They must never contain credentials, access tokens, private keys, or raw customer data.

The safe repeatability gate is:

```bash
pnpm acceptance:repeatability -- --repository wazootech/wazoo-console --issue 84 --base-sha <full-40-character-sha> --worktree /tmp/factory-acceptance --disposable true
```

It writes redacted evidence under `.eve/acceptance-repeatability/`, performs two fixture replays, requires equal canonical traces, and records zero model calls, GitHub mutations, branch pushes, or pull requests. It is not a live-provider test. The write-capable live eval remains opt-in and additionally requires `EVE_LIVE_ACCEPTANCE_CONFIRM=1`, `EVE_LIVE_ACCEPTANCE_TARGET=wazootech/wazoo-console`, `EVE_LIVE_ACCEPTANCE_ISSUE=84`, `EVE_LIVE_ACCEPTANCE_BASE_SHA=<full SHA>`, and `EVE_LIVE_ACCEPTANCE_DISPOSABLE=1`.

The separate [computer-memory](https://github.com/wazootech/computer-memory) repository is a follow-up integration point, tracked separately from the first Blob-backed factory implementation.

User preferences and transient station handoff artifacts remain in their isolated storage namespaces. They are not the Computer factory brain.

## Access and configuration

The authenticated web chat is served at `/`. In development:

```bash
pnpm dev
```

The Eve runtime can also be exercised with:

```bash
pnpm dev:eve
```

Production deployment uses:

```bash
eve deploy
```

The production runtime needs `BETTER_AUTH_SECRET`, `VERCEL_APP_CLIENT_ID`, `VERCEL_APP_CLIENT_SECRET`, `GITHUB_APP_ID`, `GITHUB_APP_INSTALLATION_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_WEBHOOK_SECRET`, `FACTORY_APPROVAL_SECRET`, and `AI_GATEWAY_API_KEY`. The Discord channel needs none of these: it lives in `wazootech/computer` and runs on the Zo host. Two optional variables, `GITHUB_OAUTH_CLIENT_ID` and `GITHUB_OAUTH_CLIENT_SECRET`, enable GitHub sign-in; set both or neither. When they are set, a signed-in person can link their GitHub account, and that verified login is what attributes a write approval from chat.

## Session repository attachment

The GitHub channel stamps the repository from the verified event. A session a person starts from the web chat or the Eve HTTP API had no such stamp, so every `github__*` tool stayed unbound and work stopped at "No verified GitHub repository is attached to this session". The Eve channel now attaches a repository at request time: it resolves `owner/name` through the App installation token and stamps the same `githubRepository*` attributes the GitHub channel uses, so the tool binders and `preflight` need no special case.

- The default is `COMPUTER_SESSION_REPOSITORY`, falling back to `wazootech/workspace` (the federation manifest repo, which lists every repository in the org).
- A request can point the session at another repository with the `x-computer-repository: owner/name` header. A malformed value fails closed.
- Coverage is proven, not assumed: `GET /repos/{owner}/{name}` succeeds only when the installation can reach the repository, and a repository the App cannot see is reported (`not-covered`) instead of binding tools that would fail later.
- Only human principals are attached. App and runtime principals (eval and schedule runs) keep the auth they have today, so unattended runs do not gain a GitHub write surface.
- `preflight` reports the repository it checked, the App installation coverage for it, and the token's permission set; pass `repository` to check another `owner/name`.

The GitHub channel uses Eve's native GitHub App authentication with the WazooComputer App credentials. It does not require `GITHUB_CONNECTOR`. The GitHub webhook secret authenticates inbound events; the App private key and installation ID authorize GitHub API calls and sandbox egress.
A direct `@wazootech/computer` invocation is accepted in an issue or pull-request body when it is opened or edited, in issue and pull-request timeline comments, and in inline pull-request review comments. For timeline and review comments, `@wazootech/computer` is the native channel trigger; Eve prefilters those events by one configured `botName`. `@wazoocomputer` and `@wazoocomputer[bot]` remain compatibility aliases for body-based triggers. Direct invocations require the sender to be an active member of the configured approver team.

## Approval identity

Every GitHub write that a person asks for is approval-gated, and the approval is only accepted from a verified member of the approver team. A GitHub issue or pull-request reply carries the sender's login, so those approvals resolve directly. A chat session has no sender, so its approvals are attributed to the signed-in person's linked GitHub account: the account id must appear in the approver-team roster, and the login from that roster is stamped on the session as `githubLogin`.

To enable it, register a GitHub OAuth App with the callback URL `https://<deployment>/api/auth/callback/github`, set `GITHUB_OAUTH_CLIENT_ID` and `GITHUB_OAUTH_CLIENT_SECRET`, and sign in with GitHub once to link the account. Without the link there is no verified login, so an approval from chat is rejected with that specific reason instead of a generic one; the GitHub path keeps working throughout.

## Discord channel and persona

Computer's `@Computer` Discord channel is not part of this repository. It runs on
the Zo host as the `computer-discord` service and answers mentions through
Computer's Zo persona. The channel, its admission policy, its Zo deploy tooling,
and Computer's prompt of record (`agent/instructions.md`) live in the private
[`wazootech/computer`](https://github.com/wazootech/computer) repository, Computer's
persona repository. They moved there on 2026-11-17; their earlier history is in
this repository's log.

`agent/instructions.ts` here is the copy of that prompt the Eve app compiles in.
When it disagrees with `wazootech/computer`'s `agent/instructions.md`, that file is
the source and this one is what gets fixed.

## Validation

```bash
pnpm typecheck
pnpm test
pnpm build
pnpm build:eve
```

## Agent File

Retired 2026-09-25. This repository no longer publishes or checks an [Agent File](https://github.com/letta-ai/agent-file) projection.

The projection described less than the agent is: channels, subagents, sandboxes, approval tiers, and hooks have no `.af` counterpart, so the file could not be imported and run. It was also the only consumer of its own artifact — Computer does not run a Letta framework, and nothing read the file back.

What was removed: `agents/@wazootech/computer/computer.af`, `agent/agent-file-declaration.json`, `lib/agent-file-{schema,project,privacy,test}.ts`, `scripts/export-agent-file.ts`, the `export:agent-file` / `check:agent-file` scripts, the `check:agent-file` CI step, and the documentation that described them. The published file is preserved in `wazootech/data` at `archives/agent-file/data.af`, alongside the reason it was retired there.

Nothing about the live agent changed: this artifact was never read at runtime.

The factory evals are under `evals/`. Full-pipeline evals can create branches and consume model tokens, so run them only against a disposable target repository.
