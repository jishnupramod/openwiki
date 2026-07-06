# Subscription-agent execution engines for OpenWiki

**Date:** 2026-07-06
**Status:** Approved design, pre-implementation
**Goal quality bar:** upstream-contributable to `langchain-ai/openwiki`

## Problem

OpenWiki can only run against pay-per-token API keys. Every supported provider
(`openrouter`, `baseten`, `fireworks`, `openai`, `openai-compatible`,
`anthropic`) is a LangChain chat model plugged into a DeepAgents loop, and
`runOpenWikiAgent()` hard-fails without the provider's API key
(`ensureProviderKey`). Many developers already pay for a subscription coding
agent — Claude Code (Claude Pro/Max), Codex CLI (ChatGPT Plus/Pro), internal
tools such as IBM Bob — and want OpenWiki to use that entitlement instead of a
second, metered API bill.

## Decision summary

| Decision          | Choice                                                                                                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| End goal          | Design for an eventual upstream PR (clean abstraction, tests, docs)                                                                                                        |
| Run surface (v1)  | Local interactive/one-shot runs only; CI/scheduled updates stay on API keys for now, design leaves room for CI tokens later                                                |
| Reference agent   | Claude Code (headless `claude -p`)                                                                                                                                         |
| Integration style | Full delegation: the subscription agent runs the whole documentation task with its own tools                                                                               |
| Mechanism         | Subprocess adapters behind a common `AgentCliAdapter` interface (no new npm dependencies); adapter internals may later swap to a vendor SDK without changing the interface |

Explicitly **out of scope**: extracting subscription OAuth tokens to call raw
vendor APIs (ToS-violating), a "model-only" LangChain shim over an agent CLI,
CI/GitHub Actions subscription auth (`claude setup-token`) in v1, and any
vendor beyond Claude Code in the first milestone (Codex is the planned second
adapter and is used to sanity-check the interface shape).

## Architecture

OpenWiki gains a second _kind_ of provider. Shared run scaffolding — prompt
assembly, git-evidence run context, update no-op detection, doc content
snapshot, `.last-update.json` metadata — remains in OpenWiki and is common to
both kinds. Only the execution engine differs:

- `kind: "api"` (existing): LangChain model + DeepAgents loop + LocalShellBackend.
- `kind: "agent-cli"` (new): spawn a locally installed, subscription-authenticated
  agent CLI in headless mode with OpenWiki's prompt; stream its JSON events back
  into the existing UI.

### 1. Provider model — `src/constants.ts`

- `ProviderConfig` becomes a discriminated union:
  - `ApiProviderConfig` (existing fields: `apiKeyEnvKey`, `baseURL`,
    `baseUrlEnvKey`, `requiresBaseUrl`, `label`, `modelOptions`).
  - `AgentCliProviderConfig`: `{ kind: "agent-cli"; label; modelOptions;
adapterId; binaryEnvKey }` — no API key, no base URL.
- New provider id `claude-code`, added to `OpenWikiProvider`,
  `SELECTABLE_OPENWIKI_PROVIDERS`, and `PROVIDER_CONFIGS`:
  - Label: `Claude Code (subscription)`.
  - Model options: `default` (omit `--model`), `sonnet`, `opus`, `haiku`.
  - `binaryEnvKey: "OPENWIKI_CLAUDE_CODE_BINARY"` — optional path override for
    wrapped/internal binaries.
- Helper `isAgentCliProvider(provider)` used by runtime and onboarding.
- Existing helpers (`getProviderApiKeyEnvKey`, `providerRequiresBaseUrl`, …)
  narrow to the API branch; call sites are updated rather than made nullable-at-a-distance.

### 2. Engine module — `src/agent/engines/`

- `types.ts`:

  ```ts
  type EngineRunSpec = {
    command: OpenWikiCommand;
    cwd: string;
    modelId: string; // "default" => adapter omits model flag
    prompt: string; // fully assembled user prompt
    systemPrompt: string; // appended, not replacing the agent's own
    resumeSessionId?: string; // interactive follow-ups
  };

  type AgentCliAdapter = {
    id: "claude-code"; // union grows with each adapter
    defaultBinary: string; // "claude"
    binaryEnvKey: string; // override, e.g. OPENWIKI_CLAUDE_CODE_BINARY
    installHint: string; // shown when binary/auth missing
    detectInstall(
      binary: string,
    ): Promise<{ found: boolean; version?: string }>;
    buildArgs(spec: EngineRunSpec): string[];
    parseEvent(line: unknown): AgentCliEvent | null;
  };
  ```

  `AgentCliEvent` is a thin union: `{ type: "openwiki"; event: OpenWikiRunEvent }`,
  `{ type: "session"; sessionId: string }`, `{ type: "result"; ok: boolean;
errorMessage?: string }`.

- `runner.ts` (adapter-agnostic):
  - Resolves the binary (env override → default), verifies it exists on PATH.
  - Spawns with `cwd` = repository root, prompt written to stdin, stdout parsed
    line-by-line as NDJSON through `adapter.parseEvent`.
  - Forwards mapped `OpenWikiRunEvent`s to `options.onEvent` (text, tool_start,
    tool_end, debug) so `src/cli.tsx` renders delegated runs unchanged.
  - Captures the session id for follow-up resumption.
  - Enforces an overall run timeout (default 30 minutes,
    `OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS` to override), kills the process group
    on timeout/abort, and retains a bounded stderr tail for error reporting.

- `claude-code.ts` (reference adapter):
  - Invocation: `claude -p --output-format stream-json --verbose
--permission-mode acceptEdits --append-system-prompt <system prompt>
[--model <model>] [--resume <sessionId>] --allowedTools <list>`.
  - Allowed tools (conservative, documentation-shaped): `Read`, `Glob`, `Grep`,
    `LS`, `Write`, `Edit`, `MultiEdit`, plus read-only git via
    `Bash(git log:*)`, `Bash(git diff:*)`, `Bash(git show:*)`,
    `Bash(git status:*)`. Exact strings validated against the installed CLI
    during implementation.
  - Event mapping from Claude Code stream-json: `system/init` → session id +
    debug; `assistant` text deltas → `{type:"text"}`; `tool_use` →
    `{type:"tool_start"}` (reusing the existing `formatToolArgs` display
    conventions); tool results → `{type:"tool_end"}`; `result` → terminal
    success/error.

### 3. Runtime branch — `src/agent/index.ts`

- After `resolveConfiguredProvider()`: if `isAgentCliProvider(provider)`, skip
  `ensureProviderKey`/`ensureProviderBaseUrl` and dispatch to
  `runAgentCliEngine(command, cwd, options, provider, modelId)`.
- The engine path reuses, in the same order as the API path:
  `loadOpenWikiEnv`, update no-op check, `createRunContext`,
  `createOpenWikiContentSnapshot` before/after, and
  `writeLastUpdateMetadata` when docs changed.
- Prompt assembly: same `createSystemPrompt(command)` / `createUserPrompt(...)`
  content, but the runtime-notes block in `createRunUserMessage` is
  engine-aware — the DeepAgents "virtual filesystem" instructions are replaced
  with: work directly in the repository root using real relative paths; do not
  touch paths outside it.
- Follow-ups: the first run of an interactive session stores the reported
  session id keyed by OpenWiki thread id (in-memory, same lifetime as today's
  `threadId` usage in `cli.tsx`); follow-up messages pass `resumeSessionId`.
- OpenRouter fallback routing and the debug-fetch wrapper remain API-only.
- `OpenWikiRunResult` unchanged (`{ command, model, skipped? }`), so `cli.tsx`
  and metadata writes need no structural changes.

### 4. Onboarding — `src/credentials.tsx`

- Provider select gains `Claude Code (subscription)`.
- For agent-cli providers the `api-key` / `base-url` steps are replaced by an
  install check step: run `detectInstall`; on success show the detected
  version and continue to model selection; on failure show `installHint`
  (install command + "run `claude` once and complete the login") and let the
  user retry or pick another provider.
- `needsCredentialSetup()` branches on provider kind: agent-cli providers need
  setup only when `OPENWIKI_PROVIDER` or the model id is unset.
- The LangSmith step is skipped for agent-cli providers with a one-line notice
  (delegated runs do not flow through LangChain tracing).
- `~/.openwiki/.env` stores only `OPENWIKI_PROVIDER` and `OPENWIKI_MODEL_ID`
  for these providers — no secrets; vendor auth stays in the vendor's own
  store (e.g. `~/.claude`).
- `getCredentialDiagnostics()` reports the provider, model, and binary
  override key so `--doctor`-style output stays truthful.

## Error handling

- **Binary missing** (onboarding and run time): actionable message with
  `installHint`; run aborts before any snapshot/metadata work.
- **Auth expired / not logged in**: surfaced from the CLI's error result event;
  message tells the user to run `claude` and complete login. No retry loop.
- **Non-zero exit or malformed NDJSON**: fail the run with the stderr tail and
  a debug-mode dump of the last unparsed line; never write update metadata.
- **Timeout**: kill process group, report elapsed time and the timeout env
  override.
- Model fallback (OpenRouter-style retry routes) intentionally does not apply.

## Security & ToS posture

- The delegated agent edits only the user's working tree — the same trust
  model as the existing `LocalShellBackend` (which already executes shell
  commands on the host).
- `--permission-mode acceptEdits` + explicit `--allowedTools` keeps the
  delegated agent narrower than a default interactive Claude Code session.
- Headless `claude -p` under a Pro/Max login is ordinary Claude Code usage and
  within Anthropic's terms. The design never extracts or proxies subscription
  OAuth tokens to raw APIs, and the spec documents this as a hard boundary for
  future adapters.

## Testing

Follow existing patterns in `test/` (unit-level, no live model calls):

- `constants` — provider-union narrowing, `isAgentCliProvider`, model options,
  provider normalization for `claude-code`.
- `engines/claude-code` — `buildArgs` for each command/model/resume
  combination; `parseEvent` against NDJSON fixtures captured from a real
  `claude -p --output-format stream-json` run (init, text, tool_use, result,
  error variants).
- `engines/runner` — spawn wiring with a stub executable fixture: event
  forwarding order, session capture, timeout kill, stderr-tail reporting.
- `credentials` — `needsCredentialSetup` branching and step sequence for
  agent-cli providers.
- Manual verification milestone: `openwiki --init` with
  `OPENWIKI_PROVIDER=claude-code` on a small fixture repository produces
  `openwiki/` docs and `.last-update.json`, and interactive follow-ups resume
  the same session.

## Documentation

- README: new "Use your coding-agent subscription" section (setup, model
  choices, local-only caveat, pointer for adding new adapters).
- `openwiki/` wiki: update quickstart + architecture notes that currently say
  provider support lives entirely in `PROVIDER_CONFIGS` and the
  model-creation branch, to describe the two provider kinds and the engines
  module.
- `examples/openwiki-update.yml` untouched in v1; README notes that scheduled
  updates still require an API-key provider.

## Milestones

1. Provider-kind union + `claude-code` provider entry (no behavior change for
   API providers).
2. Engines module: runner + Claude Code adapter + unit tests.
3. Runtime branch in `runOpenWikiAgent` + engine-aware prompt runtime notes.
4. Onboarding flow + diagnostics.
5. Docs + manual end-to-end verification.
6. (Later, separate specs) Codex adapter; CI subscription tokens.
