# IBM Bob (Bob Shell) adapter for OpenWiki subscription engines

**Date:** 2026-07-06
**Status:** Approved design, pre-implementation
**Depends on:** `docs/superpowers/specs/2026-07-06-subscription-agent-engines-design.md` (the `agent-cli` provider kind and engines module, shipped as fork PR #1)

## Problem

OpenWiki's `agent-cli` provider kind ships with a single adapter, `claude-code`.
IBM Bob is a publicly available AI coding agent (https://bob.ibm.com) whose CLI,
Bob Shell (`bob`), supports headless one-shot runs with NDJSON streaming — the
same delegation shape the engines module was built for. Users with a Bob
entitlement should be able to run OpenWiki documentation jobs through it with
no API key. A second adapter also pressure-tests the `AgentCliAdapter`
interface ahead of the eventual upstream PR to `langchain-ai/openwiki`.

## Decision summary

| Decision            | Choice                                                                                              |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| Provider id / label | `ibm-bob` / `IBM Bob (subscription)`                                                                |
| Binary              | `bob` by default; `OPENWIKI_IBM_BOB_BINARY` overrides                                               |
| Chat mode           | Pinned `--chat-mode advanced` (deterministic across users; Bob's most capable mode)                  |
| Model options       | `default` only ("Subscription default", omit `-m`); any hand-set `OPENWIKI_MODEL_ID` passes through |
| System prompt       | No Bob flag exists — adapter prepends it to the stdin payload via a new optional `buildStdin` hook   |
| Write permissions   | `--approval-mode auto_edit` + scoped `--allowed-tools` (mirrors Claude's `acceptEdits` + allowlist)  |
| Session resume      | `--resume <uuid>` with the `session_id` captured from the `init` event                              |

Explicitly **out of scope**: Bob's API-key mode (`BOBSHELL_API_KEY`) as a CI
path (future work, mirrors the deferred `claude setup-token` design), MCP
server passthrough, custom Bob modes, sandbox flags, and `--max-coins`
budgeting.

## Vendor contract (verified against installed Bob Shell v1.0.3 bundle + docs)

Invocation:

- Prompt is piped on **stdin** (documented: `cat prompt.txt | bob`); the
  positional prompt and deprecated `-p` are appended to stdin input. A
  non-TTY stdin runs one-shot non-interactive.
- `--output-format stream-json` emits NDJSON events on stdout.
- `--approval-mode auto_edit` auto-approves edit tools; the default headless
  mode only permits non-destructive tools; `--yolo` approves everything.
- `--allowed-tools` (array, comma-split) supports prefix scoping, e.g.
  `run_shell_command(git log)` — confirmed by the bundle's own settings docs.
- `-m/--model <string>`: catalog is served from Bob's backend; omit for the
  subscription default.
- `--resume {number}|{uuid}|latest` — uuid form confirmed in bundle error text.

stream-json event schema (extracted verbatim from the bundle's emitters):

| Event                 | Shape                                                                                                    |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| `init`                | `{type:"init", timestamp, session_id, model}`                                                            |
| `message` (user echo) | `{type:"message", timestamp, role:"user", content}`                                                      |
| `message` (assistant) | `{type:"message", timestamp, role:"assistant", content, delta:true}` — streamed text deltas              |
| `tool_use`            | `{type:"tool_use", timestamp, tool_name, tool_id, parameters}`                                           |
| `tool_result`         | `{type:"tool_result", timestamp, tool_id, status:"success"\|"error", output, error?:{type,message}}`     |
| `error`               | `{type:"error", timestamp, severity:"warning"\|"error", message}` (loop detected, max session turns)     |
| `result`              | `{type:"result", timestamp, status:"success"\|"error", error?:{type,message}, stats}`                    |

Constraints:

- **No system-prompt flag.** Bob's instruction files (`.bobrules`,
  `AGENTS.md`) are repo-persistent — wrong vehicle for a per-run prompt. The
  adapter composes `systemPrompt + "\n\n" + prompt` as the stdin payload.
- **Folder trust gates approval modes.** The bundle throws
  `"Cannot enable privileged approval modes in an untrusted folder"` for any
  non-default `--approval-mode` (and disables yolo) when the workspace is
  untrusted and the trust feature is active. Trusting the target repository in
  Bob is a documented prerequisite; an untrusted run fails fast with Bob's own
  message in the stderr tail.
- **Auth.** Expired/missing SSO login fails headless with
  `BFF authentication failed … Authentication timeout` on stderr; the runner's
  existing `/login|api key|authenticat/i` hint regex matches, so the
  installHint is surfaced. `bob --version` works unauthenticated
  (safe for `detectInstall`).
- Tool names observed in the bundle mix Roo-style (`write_to_file`,
  `apply_diff`, `insert_content`, `execute_command`, `attempt_completion`) and
  gemini-style (`read_file`, `glob`, `search_file_content`,
  `run_shell_command`); exact runtime names must be confirmed from a live
  `tool_use` capture during verification.
- Writes are confined to the directory Bob was started in (documented) — the
  same working-directory-boundary posture the Claude Code adapter relies on.

## Architecture

The feature is config-driven end-to-end: `src/agent/index.ts`, `src/cli.tsx`,
`src/credentials-flow.ts`, `src/credentials.tsx`, and `src/agent/prompt.ts`
need **no changes**. The work is one new adapter plus registrations:

1. **`src/constants.ts`** — `IBM_BOB_BINARY_ENV_KEY = "OPENWIKI_IBM_BOB_BINARY"`;
   `"ibm-bob"` added to `OpenWikiProvider`, `SELECTABLE_OPENWIKI_PROVIDERS`,
   and `PROVIDER_CONFIGS` (kind `agent-cli`, defaultBinary `"bob"`,
   installHint covering install one-liner, IBMid login, and folder trust;
   modelOptions `[{ id: "default", label: "Subscription default" }]`).
2. **`src/agent/engines/types.ts`** — widen `AgentCliAdapter.id` to
   `"claude-code" | "ibm-bob"`; add optional
   `buildStdin?(spec: EngineRunSpec): string` (default: `spec.prompt`).
3. **`src/agent/engines/runner.ts`** — write
   `adapter.buildStdin?.(spec) ?? spec.prompt` to the child's stdin; no other
   changes (timeout, stderr tail, session map, orphan cleanup are already
   adapter-agnostic).
4. **`src/agent/engines/ibm-bob.ts`** (new) — `BOB_ALLOWED_TOOLS` (read-only
   git via `run_shell_command(git …)` prefixes plus
   `run_shell_command(rm -f openwiki/_plan.md)`; edit tools come from
   `auto_edit`; network tools stay unapproved), `buildArgs`, `buildStdin`,
   `detectInstall` (`--version`, 15s timeout), and `parseEvent` mapping the
   schema above onto `AgentCliEvent` (`init` → session + debug; assistant
   deltas → text; user echo → `[]`; `tool_use` → tool_start; `tool_result` →
   tool_end; `error` → debug; `result` → terminal ok/error).
5. **`src/agent/engines/index.ts`** — register `"ibm-bob": ibmBobAdapter`.
6. **`src/env.ts`** — add the binary env key to imports, `managedEnvKeys`,
   `getCredentialDiagnostics`, and `isNonSecretDiagnosticKey`.

## Error handling

Same posture as the Claude Code adapter — the runner already covers binary
missing (installHint), auth failure (stderr regex → installHint), non-zero
exit/malformed NDJSON (stderr tail, no metadata write), and timeout (process
group kill). Bob-specific: the untrusted-folder error arrives verbatim in the
stderr tail; README documents the trust prerequisite.

## Security & ToS posture

Headless `bob` under the user's own IBMid login is ordinary Bob Shell usage.
No tokens are extracted or proxied (hard boundary, unchanged). The delegated
agent is narrower than a default interactive Bob session: pinned mode,
`auto_edit` instead of yolo, explicit shell allowlist, and Bob's own
cwd-confinement for writes.

## Testing

- `test/ibm-bob-adapter.test.ts` (new): `buildArgs` matrix (default/named
  model, resume), `buildStdin` composition, `detectInstall`, `parseEvent`
  against fixture lines matching the bundle-extracted schema, allowlist shape.
- `test/provider-kinds.test.ts`: extend with `ibm-bob` config fields,
  selectability, and diagnostics assertions.
- `test/agent-cli-runner.test.ts`: registry returns the Bob adapter; a
  stdin-composition test through the runner via a stub that echoes stdin.
- Manual verification milestone (requires a live `bob` login and a trusted
  fixture repo): capture a real stream-json run and diff field-for-field
  against fixtures; confirm runtime tool names for the allowlist; confirm
  `auto_edit` writes headless in a trusted folder (decision point: fall back
  to `--yolo` if not); confirm stdin-only one-shot and `--resume <uuid>`
  continuity; end-to-end `--init` + `--update` with
  `OPENWIKI_PROVIDER=ibm-bob`; negative paths (untrusted folder, logged out).

## Documentation

- README "Use your coding-agent subscription" section gains IBM Bob setup
  (provider switch, login, folder-trust prerequisite,
  `OPENWIKI_IBM_BOB_BINARY`).
- `openwiki/` pages that say "currently `claude-code`" list both adapters
  (`cli/usage.md`, `quickstart.md`, `agent/workflow.md`,
  `architecture/overview.md`, `operations/credentials-and-updates.md`).

## Milestones

1. Interface extension (`buildStdin`) + runner line + `ibm-bob` provider
   entry and registrations (no behavior change for existing providers).
2. `ibm-bob` adapter + unit tests.
3. Docs.
4. Live verification against the installed Bob Shell; adjust
   `BOB_ALLOWED_TOOLS`/approval mode from observed behavior.
