# Subscription-Agent Execution Engines Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let OpenWiki delegate documentation runs to a locally installed, subscription-authenticated coding agent CLI (Claude Code first) instead of requiring a pay-per-token API key.

**Architecture:** A new provider kind `agent-cli` joins the existing API providers in `src/constants.ts`. A new `src/agent/engines/` module spawns the vendor CLI headless (`claude -p --output-format stream-json`), parses its NDJSON events into the existing `OpenWikiRunEvent` stream, and `runOpenWikiAgent` branches to it — reusing OpenWiki's prompt assembly, git-evidence context, update no-op check, doc snapshot, and `.last-update.json` metadata unchanged.

**Tech Stack:** TypeScript (ESM, `.js` import suffixes), Node `child_process.spawn` + `readline`, Ink (onboarding UI), vitest.

**Spec:** `docs/superpowers/specs/2026-07-06-subscription-agent-engines-design.md`

## Global Constraints

- Node >= 20, pnpm, `"type": "module"`; internal imports use `.js` suffixes even from `.ts`/`.tsx` files (tests import `../src/<file>.ts` directly, matching `test/update-noop.test.ts`).
- **No new npm dependencies.**
- Behavior for the six existing API providers must be unchanged (their `PROVIDER_CONFIGS` entries only gain `kind: "api"`).
- New provider id is exactly `claude-code`; its label is exactly `Claude Code (subscription)`; its binary override env key is exactly `OPENWIKI_CLAUDE_CODE_BINARY`; run timeout override env key is exactly `OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS` (default 1800 seconds).
- Model options for `claude-code`: `default` (omit `--model`), `sonnet`, `opus`, `haiku`.
- No secrets are ever written to `~/.openwiki/.env` for agent-cli providers (only `OPENWIKI_PROVIDER`, `OPENWIKI_MODEL_ID`).
- Verification commands: `pnpm test` (vitest run), `pnpm run lint:check`, `pnpm run build`. Run a single test file with `pnpm vitest run test/<file>.test.ts`.
- Work happens on branch `feat/subscription-agent-engines`. Commit after every task (green tests + lint).

---

### Task 1: Provider kinds in constants.ts and env diagnostics

**Files:**

- Modify: `src/constants.ts`
- Modify: `src/env.ts:36-50` (managedEnvKeys), `src/env.ts:74-92` (getCredentialDiagnostics), `src/env.ts:176-183` (isNonSecretDiagnosticKey)
- Test: `test/provider-kinds.test.ts`

**Interfaces:**

- Consumes: nothing new.
- Produces (used by every later task):
  - `type OpenWikiProvider` now includes `"claude-code"`.
  - `CLAUDE_CODE_BINARY_ENV_KEY = "OPENWIKI_CLAUDE_CODE_BINARY"` (exported const).
  - `type AgentCliProviderConfig = { kind: "agent-cli"; binaryEnvKey: string; defaultBinary: string; installHint: string; label: string; modelOptions: ProviderModelOption[] }` (exported type).
  - `isAgentCliProvider(provider: OpenWikiProvider): boolean` (exported).
  - `getAgentCliProviderConfig(provider: OpenWikiProvider): AgentCliProviderConfig` (exported, throws for API providers).
  - `getProviderApiKeyEnvKey(provider)` now **throws** for agent-cli providers.
  - `resolveProviderBaseUrl` / `getProviderBaseUrlEnvKey` return `undefined` and `providerRequiresBaseUrl` returns `false` for agent-cli providers.
  - `claude-code` is **NOT** added to `SELECTABLE_OPENWIKI_PROVIDERS` yet (that happens in Task 8, once the onboarding UI can handle it); it is reachable via `OPENWIKI_PROVIDER=claude-code`.

- [ ] **Step 1: Write the failing test**

Create `test/provider-kinds.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import {
  CLAUDE_CODE_BINARY_ENV_KEY,
  getAgentCliProviderConfig,
  getDefaultModelId,
  getProviderApiKeyEnvKey,
  getProviderLabel,
  isAgentCliProvider,
  isValidModelId,
  normalizeProvider,
  providerRequiresBaseUrl,
  resolveProviderBaseUrl,
} from "../src/constants.ts";
import { getCredentialDiagnostics } from "../src/env.ts";

describe("agent-cli provider kinds", () => {
  test("claude-code is a valid provider id", () => {
    expect(normalizeProvider("claude-code")).toBe("claude-code");
    expect(normalizeProvider("CLAUDE-CODE")).toBe("claude-code");
  });

  test("isAgentCliProvider distinguishes provider kinds", () => {
    expect(isAgentCliProvider("claude-code")).toBe(true);
    expect(isAgentCliProvider("anthropic")).toBe(false);
    expect(isAgentCliProvider("openrouter")).toBe(false);
    expect(isAgentCliProvider("openai-compatible")).toBe(false);
  });

  test("agent-cli config exposes binary, override key, and install hint", () => {
    const config = getAgentCliProviderConfig("claude-code");

    expect(config.kind).toBe("agent-cli");
    expect(config.defaultBinary).toBe("claude");
    expect(config.binaryEnvKey).toBe(CLAUDE_CODE_BINARY_ENV_KEY);
    expect(config.installHint).toContain("claude");
  });

  test("getAgentCliProviderConfig rejects API providers", () => {
    expect(() => getAgentCliProviderConfig("openai")).toThrow(/openai/);
  });

  test("model options start with the subscription default", () => {
    expect(getDefaultModelId("claude-code")).toBe("default");
    expect(isValidModelId("default")).toBe(true);
  });

  test("api-key helper rejects agent-cli providers", () => {
    expect(() => getProviderApiKeyEnvKey("claude-code")).toThrow(/claude-code/);
  });

  test("base URL helpers treat agent-cli providers as endpoint-free", () => {
    expect(providerRequiresBaseUrl("claude-code")).toBe(false);
    expect(resolveProviderBaseUrl("claude-code")).toBeUndefined();
  });

  test("label reads as a subscription provider", () => {
    expect(getProviderLabel("claude-code")).toBe("Claude Code (subscription)");
  });

  test("credential diagnostics include the claude-code binary override", async () => {
    const diagnostics = await getCredentialDiagnostics();

    expect(diagnostics.map((diagnostic) => diagnostic.key)).toContain(
      CLAUDE_CODE_BINARY_ENV_KEY,
    );
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run test/provider-kinds.test.ts`
Expected: FAIL — `"claude-code"` is not a valid `OpenWikiProvider` / `CLAUDE_CODE_BINARY_ENV_KEY` has no export (TypeScript/module errors count as the failing state).

- [ ] **Step 3: Implement in `src/constants.ts`**

Add after line 10 (`export const OPENROUTER_API_KEY_ENV_KEY ...`):

```ts
export const CLAUDE_CODE_BINARY_ENV_KEY = "OPENWIKI_CLAUDE_CODE_BINARY";
```

Change the provider union (keep alphabetical):

```ts
export type OpenWikiProvider =
  | "anthropic"
  | "baseten"
  | "claude-code"
  | "fireworks"
  | "openai"
  | "openai-compatible"
  | "openrouter";
```

Replace the single `ProviderConfig` type with a discriminated union (export the two branch types):

```ts
export type ApiProviderConfig = {
  kind: "api";
  apiKeyEnvKey: string;
  baseURL?: string;
  /**
   * Environment variable that, when set, overrides {@link ApiProviderConfig.baseURL}
   * with an alternative base URL (e.g. a self-hosted or proxied endpoint).
   */
  baseUrlEnvKey?: string;
  /**
   * When true, the provider has no default endpoint and requires a base URL to
   * be supplied via {@link ApiProviderConfig.baseUrlEnvKey}.
   */
  requiresBaseUrl?: boolean;
  label: string;
  modelOptions: ProviderModelOption[];
};

export type AgentCliProviderConfig = {
  kind: "agent-cli";
  /** Environment variable that overrides the default binary path. */
  binaryEnvKey: string;
  defaultBinary: string;
  /** Shown when the binary is missing or not logged in. */
  installHint: string;
  label: string;
  modelOptions: ProviderModelOption[];
};

type ProviderConfig = ApiProviderConfig | AgentCliProviderConfig;
```

Add `kind: "api",` as the first property of all six existing `PROVIDER_CONFIGS` entries, and add the new entry (place it after `anthropic`):

```ts
  "claude-code": {
    kind: "agent-cli",
    binaryEnvKey: CLAUDE_CODE_BINARY_ENV_KEY,
    defaultBinary: "claude",
    installHint:
      "Install Claude Code (npm install -g @anthropic-ai/claude-code), then run `claude` once and complete the subscription login.",
    label: "Claude Code (subscription)",
    modelOptions: [
      { id: "default", label: "Subscription default" },
      { id: "sonnet", label: "Sonnet" },
      { id: "opus", label: "Opus" },
      { id: "haiku", label: "Haiku" },
    ],
  },
```

Do **not** touch `SELECTABLE_OPENWIKI_PROVIDERS` in this task. Because that array is typed `readonly SelectableOpenWikiProvider[]` and `SelectableOpenWikiProvider = OpenWikiProvider`, it still compiles.

Replace the four provider-shape helpers:

```ts
export function isAgentCliProvider(provider: OpenWikiProvider): boolean {
  return getProviderConfig(provider).kind === "agent-cli";
}

export function getAgentCliProviderConfig(
  provider: OpenWikiProvider,
): AgentCliProviderConfig {
  const config = getProviderConfig(provider);

  if (config.kind !== "agent-cli") {
    throw new Error(`${provider} is not an agent CLI provider.`);
  }

  return config;
}

function getApiProviderConfig(provider: OpenWikiProvider): ApiProviderConfig {
  const config = getProviderConfig(provider);

  if (config.kind !== "api") {
    throw new Error(
      `${provider} is an agent CLI provider and has no API key configuration.`,
    );
  }

  return config;
}

export function getProviderApiKeyEnvKey(provider: OpenWikiProvider): string {
  return getApiProviderConfig(provider).apiKeyEnvKey;
}

export function resolveProviderBaseUrl(
  provider: OpenWikiProvider,
  env: NodeJS.ProcessEnv = process.env,
): string | undefined {
  const config = getProviderConfig(provider);

  if (config.kind !== "api") {
    return undefined;
  }

  const override = config.baseUrlEnvKey ? env[config.baseUrlEnvKey] : undefined;
  const trimmedOverride = override?.trim();

  if (trimmedOverride) {
    return trimmedOverride;
  }

  return config.baseURL;
}

export function getProviderBaseUrlEnvKey(
  provider: OpenWikiProvider,
): string | undefined {
  const config = getProviderConfig(provider);

  return config.kind === "api" ? config.baseUrlEnvKey : undefined;
}

export function providerRequiresBaseUrl(provider: OpenWikiProvider): boolean {
  const config = getProviderConfig(provider);

  return config.kind === "api" && config.requiresBaseUrl === true;
}
```

(Keep the existing doc comment on `resolveProviderBaseUrl`.)

- [ ] **Step 4: Implement in `src/env.ts`**

Add `CLAUDE_CODE_BINARY_ENV_KEY` to the import from `./constants.js`. Then:

- In `managedEnvKeys`, add `CLAUDE_CODE_BINARY_ENV_KEY,` after `OPENROUTER_API_KEY_ENV_KEY,`.
- In `getCredentialDiagnostics()`, add `createCredentialDiagnostic(CLAUDE_CODE_BINARY_ENV_KEY, fileEnv),` after the `OPENROUTER_API_KEY_ENV_KEY` line.
- In `isNonSecretDiagnosticKey()`, add `key === CLAUDE_CODE_BINARY_ENV_KEY ||` to the returned condition (a binary path is not a secret).

- [ ] **Step 5: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/provider-kinds.test.ts` — Expected: PASS (9 tests).
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green (the existing `update-noop` suite still passes; `tsc` confirms no call site broke).

- [ ] **Step 6: Commit**

```bash
git add src/constants.ts src/env.ts test/provider-kinds.test.ts
git commit -m "feat: add agent-cli provider kind with claude-code config"
```

---

### Task 2: Extract shared tool-call formatting into `src/agent/tool-format.ts`

The engines module (Task 4/5) needs the same tool-call display formatting the DeepAgents stream parser uses. Importing it from `src/agent/index.ts` would create an import cycle (index → engines → index), so move it to a leaf module.

**Files:**

- Create: `src/agent/tool-format.ts`
- Modify: `src/agent/index.ts:952-998` (delete the moved functions, import them instead)
- Test: `test/tool-format.test.ts`

**Interfaces:**

- Consumes: nothing new.
- Produces (used by Tasks 4-5 and by `src/agent/index.ts`):
  - `formatToolCallName(name: string): string`
  - `formatToolArgs(input: unknown): string`
  - `formatToolValue(value: unknown): string`
  - `createSyntheticToolCallId(name: string, input: unknown): string`

- [ ] **Step 1: Write the failing test**

Create `test/tool-format.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import {
  createSyntheticToolCallId,
  formatToolArgs,
  formatToolCallName,
} from "../src/agent/tool-format.ts";

describe("formatToolArgs", () => {
  test("formats object input as key=value pairs", () => {
    expect(formatToolArgs({ path: "README.md", limit: 5 })).toBe(
      'path="README.md", limit=5',
    );
  });

  test("parses stringified JSON input", () => {
    expect(formatToolArgs('{"a":1}')).toBe("a=1");
  });

  test("handles array, null, and undefined input", () => {
    expect(formatToolArgs(["a", 1])).toBe('"a", 1');
    expect(formatToolArgs(null)).toBe("");
    expect(formatToolArgs(undefined)).toBe("");
  });
});

describe("formatToolCallName", () => {
  test("maps execute to Execute and leaves other names as-is", () => {
    expect(formatToolCallName("execute")).toBe("Execute");
    expect(formatToolCallName("Read")).toBe("Read");
  });
});

describe("createSyntheticToolCallId", () => {
  test("derives a stable id from name and input", () => {
    expect(createSyntheticToolCallId("Read", { path: "a" })).toBe(
      'Read:{"path":"a"}',
    );
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run test/tool-format.test.ts`
Expected: FAIL — cannot resolve `../src/agent/tool-format.ts`.

- [ ] **Step 3: Create `src/agent/tool-format.ts`**

Move these functions **verbatim** from `src/agent/index.ts` (currently at lines 952-998) and add `export` to the four public ones:

```ts
export function formatToolCallName(name: string): string {
  return name === "execute" ? "Execute" : name;
}

export function formatToolArgs(input: unknown): string {
  const value = parseStringifiedJson(input);

  if (isRecord(value)) {
    return Object.entries(value)
      .map(([key, argValue]) => `${key}=${formatToolValue(argValue)}`)
      .join(", ");
  }

  if (Array.isArray(value)) {
    return value.map(formatToolValue).join(", ");
  }

  if (value === undefined || value === null) {
    return "";
  }

  return formatToolValue(value);
}

export function formatToolValue(value: unknown): string {
  if (typeof value === "string") {
    return JSON.stringify(value);
  }

  return JSON.stringify(value) ?? String(value);
}

export function createSyntheticToolCallId(
  name: string,
  input: unknown,
): string {
  return `${name}:${formatToolValue(input)}`;
}

function parseStringifiedJson(value: unknown): unknown {
  if (typeof value !== "string") {
    return value;
  }

  try {
    return JSON.parse(value);
  } catch {
    return value;
  }
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}
```

- [ ] **Step 4: Update `src/agent/index.ts`**

Delete `formatToolCallName`, `formatToolArgs`, `formatToolValue`, and `createSyntheticToolCallId` from `src/agent/index.ts` (keep its private `parseStringifiedJson` and `isRecord` — they are used elsewhere in the file). Add the import:

```ts
import {
  createSyntheticToolCallId,
  formatToolArgs,
  formatToolCallName,
} from "./tool-format.js";
```

- [ ] **Step 5: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/tool-format.test.ts` — Expected: PASS.
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 6: Commit**

```bash
git add src/agent/tool-format.ts src/agent/index.ts test/tool-format.test.ts
git commit -m "refactor: extract tool-call formatting into tool-format module"
```

---

### Task 3: Engine types and the Claude Code adapter (args + install detection)

**Files:**

- Create: `src/agent/engines/types.ts`
- Create: `src/agent/engines/claude-code.ts` (buildArgs + detectInstall; `parseEvent` returns `[]` for now)
- Test: `test/claude-code-adapter.test.ts`

**Interfaces:**

- Consumes: `OpenWikiCommand`, `OpenWikiRunEvent` from `src/agent/types.ts` (Task 0 baseline).
- Produces (used by Tasks 4-8):
  - `type EngineRunSpec = { command: OpenWikiCommand; cwd: string; modelId: string; prompt: string; systemPrompt: string; resumeSessionId?: string }`
  - `type AgentCliEvent = { type: "openwiki"; event: OpenWikiRunEvent } | { type: "session"; sessionId: string } | { type: "result"; ok: boolean; errorMessage?: string }`
  - `type AgentCliInstallStatus = { found: boolean; version?: string }`
  - `type AgentCliAdapter = { id: "claude-code"; detectInstall(binary: string): Promise<AgentCliInstallStatus>; buildArgs(spec: EngineRunSpec): string[]; parseEvent(line: unknown): AgentCliEvent[] }`
  - `claudeCodeAdapter: AgentCliAdapter` and `CLAUDE_CODE_ALLOWED_TOOLS: string` (exported const, comma-joined).

- [ ] **Step 1: Write the failing test**

Create `test/claude-code-adapter.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import {
  CLAUDE_CODE_ALLOWED_TOOLS,
  claudeCodeAdapter,
} from "../src/agent/engines/claude-code.ts";
import type { EngineRunSpec } from "../src/agent/engines/types.ts";

const baseSpec: EngineRunSpec = {
  command: "init",
  cwd: "/tmp/repo",
  modelId: "default",
  prompt: "Initialize docs.",
  systemPrompt: "You are OpenWiki.",
};

describe("claudeCodeAdapter.buildArgs", () => {
  test("builds headless stream-json args with the appended system prompt", () => {
    expect(claudeCodeAdapter.buildArgs(baseSpec)).toEqual([
      "-p",
      "--output-format",
      "stream-json",
      "--verbose",
      "--permission-mode",
      "acceptEdits",
      "--append-system-prompt",
      "You are OpenWiki.",
      "--allowedTools",
      CLAUDE_CODE_ALLOWED_TOOLS,
    ]);
  });

  test("omits --model for the subscription default and adds it otherwise", () => {
    expect(claudeCodeAdapter.buildArgs(baseSpec)).not.toContain("--model");

    const args = claudeCodeAdapter.buildArgs({ ...baseSpec, modelId: "opus" });

    expect(args[args.indexOf("--model") + 1]).toBe("opus");
  });

  test("adds --resume for follow-up sessions", () => {
    const args = claudeCodeAdapter.buildArgs({
      ...baseSpec,
      resumeSessionId: "sess-1",
    });

    expect(args[args.indexOf("--resume") + 1]).toBe("sess-1");
  });

  test("allowed tools stay documentation-shaped", () => {
    const tools = CLAUDE_CODE_ALLOWED_TOOLS.split(",");

    expect(tools).toContain("Write");
    expect(tools).toContain("Edit");
    expect(tools).toContain("Bash(git log:*)");
    expect(tools).toContain("Bash(rm -f openwiki/_plan.md)");
    expect(tools).not.toContain("WebSearch");
    expect(tools).not.toContain("WebFetch");
  });
});

describe("claudeCodeAdapter.detectInstall", () => {
  test("reports a missing binary", async () => {
    const status = await claudeCodeAdapter.detectInstall(
      "definitely-not-a-real-binary-xyz",
    );

    expect(status.found).toBe(false);
  });

  test("reports a version for an executable that prints one", async () => {
    const status = await claudeCodeAdapter.detectInstall(process.execPath);

    expect(status.found).toBe(true);
    expect(status.version).toMatch(/\d+\.\d+/);
  });
});
```

(`process.execPath` is the running `node` binary; `node --version` prints e.g. `v22.11.0`, which exercises the success path without needing Claude Code installed.)

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run test/claude-code-adapter.test.ts`
Expected: FAIL — cannot resolve the engine modules.

- [ ] **Step 3: Create `src/agent/engines/types.ts`**

```ts
import type { OpenWikiCommand, OpenWikiRunEvent } from "../types.js";

export type EngineRunSpec = {
  command: OpenWikiCommand;
  cwd: string;
  /** "default" means the vendor CLI's own default model (no model flag). */
  modelId: string;
  /** Fully assembled user prompt, delivered on stdin. */
  prompt: string;
  /** Appended to (not replacing) the vendor agent's own system prompt. */
  systemPrompt: string;
  /** Vendor session id to resume for interactive follow-ups. */
  resumeSessionId?: string;
};

export type AgentCliEvent =
  | { type: "openwiki"; event: OpenWikiRunEvent }
  | { type: "session"; sessionId: string }
  | { type: "result"; ok: boolean; errorMessage?: string };

export type AgentCliInstallStatus = {
  found: boolean;
  version?: string;
};

export type AgentCliAdapter = {
  id: "claude-code";
  detectInstall(binary: string): Promise<AgentCliInstallStatus>;
  buildArgs(spec: EngineRunSpec): string[];
  /** Parses one NDJSON line of vendor output; unknown lines return []. */
  parseEvent(line: unknown): AgentCliEvent[];
};
```

- [ ] **Step 4: Create `src/agent/engines/claude-code.ts`**

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";
import type {
  AgentCliAdapter,
  AgentCliInstallStatus,
  EngineRunSpec,
} from "./types.js";

const execFileAsync = promisify(execFile);

/**
 * Documentation-shaped tool allowlist: read/search anywhere in the repo,
 * write docs, read-only git, and the single exact rm needed to clean up the
 * temporary plan file. Network tools stay excluded on purpose.
 */
export const CLAUDE_CODE_ALLOWED_TOOLS = [
  "Task",
  "TodoWrite",
  "Read",
  "Glob",
  "Grep",
  "LS",
  "Write",
  "Edit",
  "MultiEdit",
  "Bash(git log:*)",
  "Bash(git show:*)",
  "Bash(git diff:*)",
  "Bash(git status:*)",
  "Bash(git blame:*)",
  "Bash(git rev-parse:*)",
  "Bash(rm -f openwiki/_plan.md)",
].join(",");

export const claudeCodeAdapter: AgentCliAdapter = {
  id: "claude-code",

  async detectInstall(binary: string): Promise<AgentCliInstallStatus> {
    try {
      const { stdout } = await execFileAsync(binary, ["--version"], {
        timeout: 15_000,
      });

      return { found: true, version: stdout.trim() };
    } catch {
      return { found: false };
    }
  },

  buildArgs(spec: EngineRunSpec): string[] {
    const args = [
      "-p",
      "--output-format",
      "stream-json",
      "--verbose",
      "--permission-mode",
      "acceptEdits",
      "--append-system-prompt",
      spec.systemPrompt,
      "--allowedTools",
      CLAUDE_CODE_ALLOWED_TOOLS,
    ];

    if (spec.modelId !== "default") {
      args.push("--model", spec.modelId);
    }

    if (spec.resumeSessionId) {
      args.push("--resume", spec.resumeSessionId);
    }

    return args;
  },

  parseEvent(): [] {
    return [];
  },
};
```

- [ ] **Step 5: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/claude-code-adapter.test.ts` — Expected: PASS.
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 6: Commit**

```bash
git add src/agent/engines/types.ts src/agent/engines/claude-code.ts test/claude-code-adapter.test.ts
git commit -m "feat: add agent CLI engine types and claude-code adapter args"
```

---

### Task 4: Claude Code stream-json event parsing

**Files:**

- Modify: `src/agent/engines/claude-code.ts` (replace the stub `parseEvent`)
- Test: `test/claude-code-adapter.test.ts` (append a `describe` block)

**Interfaces:**

- Consumes: `formatToolArgs` from `src/agent/tool-format.ts` (Task 2), `AgentCliEvent` (Task 3).
- Produces: `claudeCodeAdapter.parseEvent(line: unknown): AgentCliEvent[]` behavior relied on by the runner (Task 5): `session` events carry `session_id`; `tool_end` openwiki events use placeholder `name: "tool"` which the runner patches from its tool_start map.

Claude Code `--output-format stream-json` emits one JSON object per line. The shapes handled (per Anthropic's headless-mode docs; verified against a live capture in Task 10):

- `{"type":"system","subtype":"init","session_id":"...","model":"...","tools":[...],...}`
- `{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"..."},{"type":"tool_use","id":"toolu_1","name":"Read","input":{...}}]},"session_id":"..."}`
- `{"type":"user","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_1","content":"...","is_error":false}]},"session_id":"..."}`
- `{"type":"result","subtype":"success"|"error_max_turns"|"error_during_execution","is_error":false,"result":"...","session_id":"..."}`

- [ ] **Step 1: Write the failing tests**

Append to `test/claude-code-adapter.test.ts`:

```ts
describe("claudeCodeAdapter.parseEvent", () => {
  test("system init yields a session event and a debug event", () => {
    const events = claudeCodeAdapter.parseEvent({
      type: "system",
      subtype: "init",
      session_id: "sess-abc",
      model: "claude-sonnet-5",
    });

    expect(events).toEqual([
      { type: "session", sessionId: "sess-abc" },
      {
        type: "openwiki",
        event: {
          type: "debug",
          message: "claude-code session initialized model=claude-sonnet-5",
        },
      },
    ]);
  });

  test("assistant text blocks become text events", () => {
    const events = claudeCodeAdapter.parseEvent({
      type: "assistant",
      message: {
        role: "assistant",
        content: [{ type: "text", text: "Working on it." }],
      },
    });

    expect(events).toEqual([
      {
        type: "openwiki",
        event: { source: "main", type: "text", text: "Working on it." },
      },
    ]);
  });

  test("assistant tool_use blocks become tool_start events", () => {
    const events = claudeCodeAdapter.parseEvent({
      type: "assistant",
      message: {
        role: "assistant",
        content: [
          { type: "text", text: "Reading." },
          {
            type: "tool_use",
            id: "toolu_1",
            name: "Read",
            input: { file_path: "README.md" },
          },
        ],
      },
    });

    expect(events).toHaveLength(2);
    expect(events[1]).toEqual({
      type: "openwiki",
      event: {
        type: "tool_start",
        call: 'Read(file_path="README.md")',
        id: "toolu_1",
        input: { file_path: "README.md" },
        name: "Read",
      },
    });
  });

  test("tool results become tool_end events with error status mapping", () => {
    const ok = claudeCodeAdapter.parseEvent({
      type: "user",
      message: {
        role: "user",
        content: [
          { type: "tool_result", tool_use_id: "toolu_1", is_error: false },
        ],
      },
    });
    const failed = claudeCodeAdapter.parseEvent({
      type: "user",
      message: {
        role: "user",
        content: [
          { type: "tool_result", tool_use_id: "toolu_2", is_error: true },
        ],
      },
    });

    expect(ok).toEqual([
      {
        type: "openwiki",
        event: {
          type: "tool_end",
          id: "toolu_1",
          name: "tool",
          status: "finished",
        },
      },
    ]);
    expect(failed[0]).toEqual({
      type: "openwiki",
      event: {
        type: "tool_end",
        id: "toolu_2",
        name: "tool",
        status: "error",
      },
    });
  });

  test("result events map success and error subtypes", () => {
    expect(
      claudeCodeAdapter.parseEvent({
        type: "result",
        subtype: "success",
        is_error: false,
        result: "done",
      }),
    ).toEqual([{ type: "result", ok: true, errorMessage: undefined }]);

    expect(
      claudeCodeAdapter.parseEvent({
        type: "result",
        subtype: "error_during_execution",
        is_error: true,
        result: "Invalid API key · Please run /login",
      }),
    ).toEqual([
      {
        type: "result",
        ok: false,
        errorMessage: "Invalid API key · Please run /login",
      },
    ]);
  });

  test("unknown lines are ignored", () => {
    expect(claudeCodeAdapter.parseEvent("not json-shaped")).toEqual([]);
    expect(claudeCodeAdapter.parseEvent({ type: "mystery" })).toEqual([]);
    expect(claudeCodeAdapter.parseEvent(null)).toEqual([]);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run test/claude-code-adapter.test.ts`
Expected: FAIL — `parseEvent` returns `[]` for everything.

- [ ] **Step 3: Implement `parseEvent`**

In `src/agent/engines/claude-code.ts`, add the import `import { formatToolArgs } from "../tool-format.js";` and `import type { AgentCliEvent } from "./types.js";` (extend the existing type import). Replace the stub `parseEvent(): []` with:

```ts
  parseEvent(line: unknown): AgentCliEvent[] {
    if (!isRecord(line) || typeof line.type !== "string") {
      return [];
    }

    if (line.type === "system") {
      return parseSystemEvent(line);
    }

    if (line.type === "assistant" && isRecord(line.message)) {
      return parseMessageContent(line.message.content, "assistant");
    }

    if (line.type === "user" && isRecord(line.message)) {
      return parseMessageContent(line.message.content, "user");
    }

    if (line.type === "result") {
      const ok = line.is_error !== true && line.subtype === "success";

      return [
        {
          type: "result",
          ok,
          errorMessage: ok
            ? undefined
            : typeof line.result === "string" && line.result.length > 0
              ? line.result
              : `Claude Code run ended with ${String(line.subtype ?? "an unknown error")}.`,
        },
      ];
    }

    return [];
  },
```

and add these module-private helpers at the bottom of the file:

```ts
function parseSystemEvent(line: Record<string, unknown>): AgentCliEvent[] {
  const events: AgentCliEvent[] = [];

  if (typeof line.session_id === "string" && line.session_id.length > 0) {
    events.push({ type: "session", sessionId: line.session_id });
  }

  if (line.subtype === "init") {
    events.push({
      type: "openwiki",
      event: {
        type: "debug",
        message: `claude-code session initialized model=${
          typeof line.model === "string" ? line.model : "unknown"
        }`,
      },
    });
  }

  return events;
}

function parseMessageContent(
  content: unknown,
  role: "assistant" | "user",
): AgentCliEvent[] {
  if (!Array.isArray(content)) {
    return [];
  }

  const events: AgentCliEvent[] = [];

  for (const block of content) {
    if (!isRecord(block)) {
      continue;
    }

    if (
      role === "assistant" &&
      block.type === "text" &&
      typeof block.text === "string" &&
      block.text.length > 0
    ) {
      events.push({
        type: "openwiki",
        event: { source: "main", type: "text", text: block.text },
      });
    }

    if (
      role === "assistant" &&
      block.type === "tool_use" &&
      typeof block.id === "string" &&
      typeof block.name === "string"
    ) {
      events.push({
        type: "openwiki",
        event: {
          type: "tool_start",
          call: `${block.name}(${formatToolArgs(block.input)})`,
          id: block.id,
          input: block.input,
          name: block.name,
        },
      });
    }

    if (
      role === "user" &&
      block.type === "tool_result" &&
      typeof block.tool_use_id === "string"
    ) {
      events.push({
        type: "openwiki",
        event: {
          type: "tool_end",
          id: block.tool_use_id,
          name: "tool",
          status: block.is_error === true ? "error" : "finished",
        },
      });
    }
  }

  return events;
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}
```

- [ ] **Step 4: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/claude-code-adapter.test.ts` — Expected: PASS.
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/engines/claude-code.ts test/claude-code-adapter.test.ts
git commit -m "feat: parse claude-code stream-json events into run events"
```

---

### Task 5: Generic runner, adapter registry, and session map

**Files:**

- Create: `src/agent/engines/runner.ts`
- Create: `src/agent/engines/index.ts`
- Test: `test/agent-cli-runner.test.ts`

**Interfaces:**

- Consumes: `AgentCliAdapter`, `EngineRunSpec`, `AgentCliEvent` (Task 3/4); `AgentCliProviderConfig` (Task 1); `OpenWikiRunOptions` from `src/agent/types.ts`.
- Produces (used by Task 7 and Task 8):
  - `runAgentCli(adapter: AgentCliAdapter, providerConfig: AgentCliProviderConfig, spec: EngineRunSpec, options: OpenWikiRunOptions): Promise<{ sessionId?: string }>` — resolves on a successful vendor result; throws with an actionable message otherwise.
  - `getThreadSessionId(threadId: string): string | undefined` / `setThreadSessionId(threadId: string, sessionId: string): void`
  - `getAgentCliAdapter(provider: OpenWikiProvider): AgentCliAdapter` (from `engines/index.ts`; throws for unregistered providers).

- [ ] **Step 1: Write the failing tests**

Create `test/agent-cli-runner.test.ts`:

```ts
import { chmod, mkdtemp, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import path from "node:path";
import { afterEach, beforeEach, describe, expect, test } from "vitest";
import {
  getAgentCliProviderConfig,
  CLAUDE_CODE_BINARY_ENV_KEY,
} from "../src/constants.ts";
import { claudeCodeAdapter } from "../src/agent/engines/claude-code.ts";
import { getAgentCliAdapter } from "../src/agent/engines/index.ts";
import {
  getThreadSessionId,
  runAgentCli,
  setThreadSessionId,
} from "../src/agent/engines/runner.ts";
import type { EngineRunSpec } from "../src/agent/engines/types.ts";
import type { OpenWikiRunEvent } from "../src/agent/types.ts";

const SUCCESS_STUB = `#!/usr/bin/env node
if (process.argv.includes("--version")) {
  console.log("0.0.0-stub");
  process.exit(0);
}
let input = "";
process.stdin.on("data", (chunk) => (input += chunk));
process.stdin.on("end", () => {
  console.log(JSON.stringify({ type: "system", subtype: "init", session_id: "stub-session", model: "stub-model" }));
  console.log(JSON.stringify({ type: "assistant", message: { role: "assistant", content: [
    { type: "text", text: "prompt-bytes:" + input.length },
    { type: "tool_use", id: "tool-1", name: "Write", input: { file_path: "openwiki/quickstart.md" } },
  ] } }));
  console.log("not-json noise line");
  console.log(JSON.stringify({ type: "user", message: { role: "user", content: [
    { type: "tool_result", tool_use_id: "tool-1", is_error: false },
  ] } }));
  console.log(JSON.stringify({ type: "result", subtype: "success", is_error: false, result: "done" }));
});
`;

const FAILURE_STUB = `#!/usr/bin/env node
if (process.argv.includes("--version")) {
  console.log("0.0.0-stub");
  process.exit(0);
}
process.stdin.resume();
process.stdin.on("end", () => {
  console.error("stderr detail: login expired");
  console.log(JSON.stringify({ type: "result", subtype: "error_during_execution", is_error: true, result: "Invalid API key" }));
  process.exit(1);
});
`;

const HANG_STUB = `#!/usr/bin/env node
if (process.argv.includes("--version")) {
  console.log("0.0.0-stub");
  process.exit(0);
}
setInterval(() => {}, 1000);
`;

async function writeStub(
  dir: string,
  name: string,
  content: string,
): Promise<string> {
  const stubPath = path.join(dir, name);
  await writeFile(stubPath, content, "utf8");
  await chmod(stubPath, 0o755);
  return stubPath;
}

const baseSpec: EngineRunSpec = {
  command: "init",
  cwd: process.cwd(),
  modelId: "default",
  prompt: "Initialize docs.",
  systemPrompt: "You are OpenWiki.",
};

let stubDir: string;
const savedEnv: Record<string, string | undefined> = {};

beforeEach(async () => {
  stubDir = await mkdtemp(path.join(tmpdir(), "openwiki-stub-"));
  for (const key of [
    CLAUDE_CODE_BINARY_ENV_KEY,
    "OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS",
  ]) {
    savedEnv[key] = process.env[key];
    delete process.env[key];
  }
});

afterEach(() => {
  for (const [key, value] of Object.entries(savedEnv)) {
    if (value === undefined) delete process.env[key];
    else process.env[key] = value;
  }
});

describe("getAgentCliAdapter", () => {
  test("returns the claude-code adapter and rejects api providers", () => {
    expect(getAgentCliAdapter("claude-code")).toBe(claudeCodeAdapter);
    expect(() => getAgentCliAdapter("openai")).toThrow(/openai/);
  });
});

describe("runAgentCli", () => {
  test("forwards events in order, patches tool_end names, and captures the session", async () => {
    process.env[CLAUDE_CODE_BINARY_ENV_KEY] = await writeStub(
      stubDir,
      "stub-ok",
      SUCCESS_STUB,
    );
    const events: OpenWikiRunEvent[] = [];

    const outcome = await runAgentCli(
      claudeCodeAdapter,
      getAgentCliProviderConfig("claude-code"),
      baseSpec,
      { onEvent: (event) => events.push(event) },
    );

    expect(outcome.sessionId).toBe("stub-session");
    const types = events.map((event) => event.type);
    expect(types).toContain("text");
    expect(types).toContain("tool_start");
    expect(types).toContain("tool_end");
    const toolEnd = events.find((event) => event.type === "tool_end");
    expect(toolEnd).toMatchObject({
      id: "tool-1",
      name: "Write",
      status: "finished",
    });
    const text = events.find((event) => event.type === "text");
    expect(text).toMatchObject({
      text: `prompt-bytes:${baseSpec.prompt.length}`,
    });
  });

  test("throws the vendor error message with the stderr tail on failure", async () => {
    process.env[CLAUDE_CODE_BINARY_ENV_KEY] = await writeStub(
      stubDir,
      "stub-fail",
      FAILURE_STUB,
    );

    await expect(
      runAgentCli(
        claudeCodeAdapter,
        getAgentCliProviderConfig("claude-code"),
        baseSpec,
        {},
      ),
    ).rejects.toThrow(/Invalid API key[\s\S]*login expired/);
  });

  test("throws an actionable install hint when the binary is missing", async () => {
    process.env[CLAUDE_CODE_BINARY_ENV_KEY] = path.join(
      stubDir,
      "does-not-exist",
    );

    await expect(
      runAgentCli(
        claudeCodeAdapter,
        getAgentCliProviderConfig("claude-code"),
        baseSpec,
        {},
      ),
    ).rejects.toThrow(/Install Claude Code/);
  });

  test("kills a hung run after the configured timeout", async () => {
    process.env[CLAUDE_CODE_BINARY_ENV_KEY] = await writeStub(
      stubDir,
      "stub-hang",
      HANG_STUB,
    );
    process.env.OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS = "1";

    await expect(
      runAgentCli(
        claudeCodeAdapter,
        getAgentCliProviderConfig("claude-code"),
        baseSpec,
        {},
      ),
    ).rejects.toThrow(/timed out after 1 seconds/);
  }, 15_000);
});

describe("thread session map", () => {
  test("stores and retrieves vendor session ids by thread id", () => {
    expect(getThreadSessionId("thread-x")).toBeUndefined();
    setThreadSessionId("thread-x", "sess-1");
    expect(getThreadSessionId("thread-x")).toBe("sess-1");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run test/agent-cli-runner.test.ts`
Expected: FAIL — cannot resolve `runner.ts` / `engines/index.ts`.

- [ ] **Step 3: Create `src/agent/engines/index.ts`**

```ts
import type { OpenWikiProvider } from "../../constants.js";
import { claudeCodeAdapter } from "./claude-code.js";
import type { AgentCliAdapter } from "./types.js";

const adapters: Partial<Record<OpenWikiProvider, AgentCliAdapter>> = {
  "claude-code": claudeCodeAdapter,
};

export function getAgentCliAdapter(
  provider: OpenWikiProvider,
): AgentCliAdapter {
  const adapter = adapters[provider];

  if (!adapter) {
    throw new Error(`No agent CLI adapter is registered for ${provider}.`);
  }

  return adapter;
}
```

- [ ] **Step 4: Create `src/agent/engines/runner.ts`**

```ts
import { spawn } from "node:child_process";
import { createInterface } from "node:readline";
import type { AgentCliProviderConfig } from "../../constants.js";
import type { OpenWikiRunOptions } from "../types.js";
import type { AgentCliAdapter, EngineRunSpec } from "./types.js";

const DEFAULT_TIMEOUT_SECONDS = 1800;
const STDERR_TAIL_LIMIT = 4000;

const threadSessionIds = new Map<string, string>();

export function getThreadSessionId(threadId: string): string | undefined {
  return threadSessionIds.get(threadId);
}

export function setThreadSessionId(threadId: string, sessionId: string): void {
  threadSessionIds.set(threadId, sessionId);
}

export type AgentCliRunOutcome = {
  sessionId?: string;
};

export async function runAgentCli(
  adapter: AgentCliAdapter,
  providerConfig: AgentCliProviderConfig,
  spec: EngineRunSpec,
  options: OpenWikiRunOptions,
): Promise<AgentCliRunOutcome> {
  const binary =
    process.env[providerConfig.binaryEnvKey]?.trim() ||
    providerConfig.defaultBinary;
  const install = await adapter.detectInstall(binary);

  if (!install.found) {
    throw new Error(
      `Could not run the ${providerConfig.label} CLI (${binary}). ${providerConfig.installHint}`,
    );
  }

  emitDebug(
    options,
    `engine=${adapter.id} binary=${binary} version=${install.version ?? "unknown"}`,
  );

  const timeoutSeconds = resolveTimeoutSeconds();
  const outcome: AgentCliRunOutcome = {};
  const toolNames = new Map<string, string>();
  let result: { ok: boolean; errorMessage?: string } | null = null;
  let stderrTail = "";
  let timedOut = false;

  const child = spawn(binary, adapter.buildArgs(spec), {
    cwd: spec.cwd,
    detached: true,
    stdio: ["pipe", "pipe", "pipe"],
  });

  const timeout = setTimeout(() => {
    timedOut = true;
    killProcessGroup(child.pid);
  }, timeoutSeconds * 1000);

  child.stdin.write(spec.prompt);
  child.stdin.end();

  child.stderr.setEncoding("utf8");
  child.stderr.on("data", (chunk: string) => {
    stderrTail = (stderrTail + chunk).slice(-STDERR_TAIL_LIMIT);
  });

  const lines = createInterface({ input: child.stdout });

  lines.on("line", (line) => {
    const trimmed = line.trim();

    if (trimmed.length === 0) {
      return;
    }

    let parsed: unknown;

    try {
      parsed = JSON.parse(trimmed);
    } catch {
      emitDebug(
        options,
        `engine.unparsedLine=${JSON.stringify(trimmed.slice(0, 200))}`,
      );
      return;
    }

    for (const event of adapter.parseEvent(parsed)) {
      if (event.type === "session") {
        outcome.sessionId = event.sessionId;
        continue;
      }

      if (event.type === "result") {
        result = { ok: event.ok, errorMessage: event.errorMessage };
        continue;
      }

      if (event.event.type === "tool_start") {
        toolNames.set(event.event.id, event.event.name);
        options.onEvent?.(event.event);
        continue;
      }

      if (event.event.type === "tool_end") {
        options.onEvent?.({
          ...event.event,
          name: toolNames.get(event.event.id) ?? event.event.name,
        });
        continue;
      }

      if (event.event.type === "debug") {
        emitDebug(options, event.event.message);
        continue;
      }

      options.onEvent?.(event.event);
    }
  });

  const exitCode = await new Promise<number | null>((resolve) => {
    child.on("close", (code) => resolve(code));
    child.on("error", () => resolve(null));
  });

  clearTimeout(timeout);

  if (timedOut) {
    throw new Error(
      `${providerConfig.label} run timed out after ${timeoutSeconds} seconds. Set OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS to allow longer runs.`,
    );
  }

  if (result?.ok) {
    return outcome;
  }

  throw new Error(
    formatRunFailure(providerConfig, result, exitCode, stderrTail),
  );
}

function formatRunFailure(
  providerConfig: AgentCliProviderConfig,
  result: { ok: boolean; errorMessage?: string } | null,
  exitCode: number | null,
  stderrTail: string,
): string {
  const summary =
    result?.errorMessage ??
    `${providerConfig.label} run failed (exit code ${exitCode ?? "unknown"}) without reporting a result.`;
  const stderr = stderrTail.trim();
  const loginHint = /login|api key|authenticat/iu.test(`${summary} ${stderr}`)
    ? ` ${providerConfig.installHint}`
    : "";

  return `${summary}${loginHint}${stderr.length > 0 ? `\nstderr: ${stderr}` : ""}`;
}

function resolveTimeoutSeconds(): number {
  const raw = process.env.OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS;
  const parsed = raw === undefined ? Number.NaN : Number.parseInt(raw, 10);

  return Number.isFinite(parsed) && parsed > 0
    ? parsed
    : DEFAULT_TIMEOUT_SECONDS;
}

function killProcessGroup(pid: number | undefined): void {
  if (pid === undefined) {
    return;
  }

  try {
    process.kill(-pid, "SIGTERM");
  } catch {
    // The process may already have exited.
  }

  setTimeout(() => {
    try {
      process.kill(-pid, "SIGKILL");
    } catch {
      // Already gone.
    }
  }, 5000).unref();
}

function emitDebug(options: OpenWikiRunOptions, message: string): void {
  if (!options.debug) {
    return;
  }

  options.onEvent?.({ type: "debug", message });
}
```

Note: `result` events being handled _before_ the tool-name patch matters — TypeScript narrows the union via the `event.type` checks in the order shown.

- [ ] **Step 5: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/agent-cli-runner.test.ts` — Expected: PASS (timeout test takes ~1-6s).
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 6: Commit**

```bash
git add src/agent/engines/runner.ts src/agent/engines/index.ts test/agent-cli-runner.test.ts
git commit -m "feat: add agent CLI runner with event mapping, timeout, and session map"
```

---

### Task 6: Engine-aware system prompt

**Files:**

- Modify: `src/agent/prompt.ts`
- Test: `test/prompt-engines.test.ts`

**Interfaces:**

- Consumes: nothing new.
- Produces (used by Task 7):
  - `export type PromptEngine = "deepagents" | "agent-cli"`
  - `createSystemPrompt(command: OpenWikiCommand, engine: PromptEngine = "deepagents"): string` — the `"deepagents"` output must be byte-identical to today's output.

- [ ] **Step 1: Write the failing tests**

Create `test/prompt-engines.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { createSystemPrompt } from "../src/agent/prompt.ts";

describe("createSystemPrompt engines", () => {
  test("deepagents variant keeps the virtual filesystem discipline", () => {
    const prompt = createSystemPrompt("init");

    expect(prompt).toContain("Use virtual paths such as /README.md");
    expect(prompt).toContain("read_file");
    expect(prompt).toContain(
      "Use /openwiki/_plan.md when writing this temporary plan",
    );
    expect(prompt).toContain(
      "When writing required documentation with filesystem tools, use /openwiki/... paths",
    );
  });

  test("agent-cli variant uses repository-relative paths and no DeepAgents tool names", () => {
    const prompt = createSystemPrompt("init", "agent-cli");

    expect(prompt).toContain("repository-relative paths");
    expect(prompt).toContain("rm -f openwiki/_plan.md");
    expect(prompt).not.toContain("read_file");
    expect(prompt).not.toContain("virtual paths");
    expect(prompt).not.toContain("/openwiki/_plan.md");
  });

  test("mode instructions are engine-independent", () => {
    expect(createSystemPrompt("update", "agent-cli")).toContain(
      "maintenance update run",
    );
    expect(createSystemPrompt("chat", "agent-cli")).toContain(
      "interactive chat turn",
    );
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run test/prompt-engines.test.ts`
Expected: FAIL — `createSystemPrompt` accepts one argument; agent-cli assertions fail.

- [ ] **Step 3: Implement in `src/agent/prompt.ts`**

Add the type and change the signature:

```ts
export type PromptEngine = "agent-cli" | "deepagents";

export function createSystemPrompt(
  command: OpenWikiCommand,
  engine: PromptEngine = "deepagents",
): string {
```

Make four surgical substitutions inside the template (leaving all other text untouched):

1. Replace the tooling sentence (line 18, starting `Use only the tools available to you.`) with `${createToolingGuidance(engine)}`.
2. Replace the first three Run discipline bullets (lines 21-23, from `- Filesystem tools are rooted...` through `...keep them inside that repository.`) with `${createPathDiscipline(engine)}`.
3. Replace the two planning-path bullets (line 41 `- Use /openwiki/_plan.md...` and line 42 `- Before completing the run, delete...`) with `${createPlanningPathNotes(engine)}`.
4. Replace line 122 (`- When writing required documentation with filesystem tools, use /openwiki/... paths, for example /openwiki/quickstart.md.`) with `${createDocPathNote(engine)}`.

Add the helpers (module-private, above `createModeInstructions`). The `deepagents` branches repeat today's text **verbatim** so the default output is unchanged:

```ts
function createToolingGuidance(engine: PromptEngine): string {
  if (engine === "agent-cli") {
    return "Use only the tools available to you. Prefer your built-in file search and read tools for discovery and targeted reads. Use git through your shell tool when it provides useful history. Do not invent files, modules, APIs, business rules, or behavior. Ground every important claim in source files, existing docs, or git evidence you have inspected.";
  }

  return "Use only the tools available to you. Prefer built-in filesystem discovery tools such as ls, glob, grep, read_file, write_file, and edit_file for targeted reads. Use git through shell execute when it provides useful history. Do not invent files, modules, APIs, business rules, or behavior. Ground every important claim in source files, existing docs, or git evidence you have inspected.";
}

function createPathDiscipline(engine: PromptEngine): string {
  if (engine === "agent-cli") {
    return `
- Your working directory is the target repository root. Use repository-relative paths such as README.md, src/..., and openwiki/quickstart.md with your file tools.
- Do not read or modify files outside the target repository.
- Shell commands run on the host. Run them from the repository root and keep them inside the repository.`.trim();
  }

  return `
- Filesystem tools are rooted at the target repository. Use virtual paths such as /README.md, /agent/..., /server/..., and /openwiki/quickstart.md with ls, read_file, write_file, edit_file, glob, and grep.
- Never pass host absolute paths like /Users/... to filesystem tools; that creates nested paths inside the repo instead of touching the intended file.
- Shell execute commands run on the host. If you use execute, run commands from the target repository directory and keep them inside that repository.`.trim();
}

function createPlanningPathNotes(engine: PromptEngine): string {
  if (engine === "agent-cli") {
    return `
- Write this temporary plan to openwiki/_plan.md with your file tools.
- Before completing the run, delete ${OPEN_WIKI_DIR}/_plan.md using your shell tool: rm -f openwiki/_plan.md.`.trim();
  }

  return `
- Use /openwiki/_plan.md when writing this temporary plan with filesystem tools.
- Before completing the run, delete ${OPEN_WIKI_DIR}/_plan.md. If there is no filesystem delete tool, use shell execute from the repository root, for example rm -f openwiki/_plan.md.`.trim();
}

function createDocPathNote(engine: PromptEngine): string {
  if (engine === "agent-cli") {
    return "- When writing required documentation, use repository-relative openwiki/... paths, for example openwiki/quickstart.md.";
  }

  return "- When writing required documentation with filesystem tools, use /openwiki/... paths, for example /openwiki/quickstart.md.";
}
```

- [ ] **Step 4: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/prompt-engines.test.ts` — Expected: PASS.
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/prompt.ts test/prompt-engines.test.ts
git commit -m "feat: add agent-cli variant of the OpenWiki system prompt"
```

---

### Task 7: Runtime dispatch in `runOpenWikiAgent` + end-to-end stub test

**Files:**

- Modify: `src/agent/index.ts:88-98` (dispatch) and add two functions near `createRunUserMessage`
- Test: `test/agent-cli-run.test.ts`

**Interfaces:**

- Consumes: `isAgentCliProvider`, `getAgentCliProviderConfig` (Task 1); `getAgentCliAdapter` (Task 5); `runAgentCli`, `getThreadSessionId`, `setThreadSessionId` (Task 5); `EngineRunSpec` (Task 3); `createSystemPrompt(command, "agent-cli")` (Task 6).
- Produces: `runOpenWikiAgent` transparently supports `OPENWIKI_PROVIDER=claude-code` — same `OpenWikiRunResult` shape, same event stream, no API key or LangSmith required. `src/cli.tsx` needs no changes.

- [ ] **Step 1: Write the failing test**

Create `test/agent-cli-run.test.ts` (repo fixture pattern mirrors `test/update-noop.test.ts`; the stub binary writes real docs so the snapshot/metadata path is exercised):

```ts
import { execFile } from "node:child_process";
import { chmod, mkdtemp, readFile, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import path from "node:path";
import { promisify } from "node:util";
import { afterEach, beforeEach, describe, expect, test } from "vitest";
import { runOpenWikiAgent } from "../src/agent/index.ts";
import type { OpenWikiRunEvent } from "../src/agent/types.ts";
import { CLAUDE_CODE_BINARY_ENV_KEY } from "../src/constants.ts";

const execFileAsync = promisify(execFile);

const DOC_WRITING_STUB = `#!/usr/bin/env node
import { mkdirSync, writeFileSync } from "node:fs";
if (process.argv.includes("--version")) {
  console.log("0.0.0-stub");
  process.exit(0);
}
let input = "";
process.stdin.on("data", (chunk) => (input += chunk));
process.stdin.on("end", () => {
  mkdirSync("openwiki", { recursive: true });
  writeFileSync("openwiki/quickstart.md", "# Stub docs\\n");
  console.log(JSON.stringify({ type: "system", subtype: "init", session_id: "stub-session" }));
  console.log(JSON.stringify({ type: "assistant", message: { role: "assistant", content: [{ type: "text", text: "Docs written." }] } }));
  console.log(JSON.stringify({ type: "result", subtype: "success", is_error: false, result: "done" }));
});
`;

async function git(cwd: string, args: string[]): Promise<string> {
  const { stdout } = await execFileAsync("git", args, { cwd });
  return stdout.trim();
}

async function createFixtureRepo(): Promise<string> {
  const repo = await mkdtemp(path.join(tmpdir(), "openwiki-agentcli-"));
  await git(repo, ["init"]);
  await git(repo, ["config", "user.email", "test@example.com"]);
  await git(repo, ["config", "user.name", "OpenWiki Test"]);
  await writeFile(path.join(repo, "README.md"), "# Fixture\n", "utf8");
  await git(repo, ["add", "."]);
  await git(repo, ["commit", "-m", "initial"]);
  return repo;
}

const ENV_KEYS = [
  "OPENWIKI_PROVIDER",
  "OPENWIKI_MODEL_ID",
  CLAUDE_CODE_BINARY_ENV_KEY,
];
const savedEnv: Record<string, string | undefined> = {};

beforeEach(async () => {
  for (const key of ENV_KEYS) {
    savedEnv[key] = process.env[key];
  }

  const stubDir = await mkdtemp(path.join(tmpdir(), "openwiki-stub-"));
  const stubPath = path.join(stubDir, "claude-stub.mjs");
  await writeFile(stubPath, DOC_WRITING_STUB, "utf8");
  await chmod(stubPath, 0o755);

  process.env.OPENWIKI_PROVIDER = "claude-code";
  process.env.OPENWIKI_MODEL_ID = "default";
  process.env[CLAUDE_CODE_BINARY_ENV_KEY] = stubPath;
});

afterEach(() => {
  for (const key of ENV_KEYS) {
    if (savedEnv[key] === undefined) delete process.env[key];
    else process.env[key] = savedEnv[key];
  }
});

describe("runOpenWikiAgent with an agent-cli provider", () => {
  test("init run delegates, streams events, writes docs and metadata without an API key", async () => {
    const repo = await createFixtureRepo();
    const events: OpenWikiRunEvent[] = [];

    const result = await runOpenWikiAgent("init", repo, {
      onEvent: (event) => events.push(event),
    });

    expect(result).toEqual({ command: "init", model: "default" });
    expect(events.some((event) => event.type === "text")).toBe(true);

    const docs = await readFile(
      path.join(repo, "openwiki", "quickstart.md"),
      "utf8",
    );
    expect(docs).toContain("Stub docs");

    const metadata = JSON.parse(
      await readFile(path.join(repo, "openwiki", ".last-update.json"), "utf8"),
    ) as { command: string; model: string };
    expect(metadata.command).toBe("init");
    expect(metadata.model).toBe("default");
  }, 30_000);

  test("chat run does not write update metadata", async () => {
    const repo = await createFixtureRepo();

    const result = await runOpenWikiAgent("chat", repo, {
      userMessage: "hello",
    });

    expect(result.command).toBe("chat");
    await expect(
      readFile(path.join(repo, "openwiki", ".last-update.json"), "utf8"),
    ).rejects.toThrow();
  }, 30_000);
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run test/agent-cli-run.test.ts`
Expected: FAIL — `runOpenWikiAgent` throws `` `${apiKeyEnvKey} is required...` `` (actually it throws from `getProviderApiKeyEnvKey`: "claude-code is an agent CLI provider and has no API key configuration."), because no dispatch exists yet.

- [ ] **Step 3: Implement the dispatch in `src/agent/index.ts`**

Add imports:

```ts
import { getAgentCliAdapter } from "./engines/index.js";
import {
  getThreadSessionId,
  runAgentCli,
  setThreadSessionId,
} from "./engines/runner.js";
import type { EngineRunSpec } from "./engines/types.js";
```

and add `getAgentCliProviderConfig, isAgentCliProvider,` to the existing `../constants.js` import list.

In `runOpenWikiAgent`, replace this block (currently lines 88-96):

```ts
const provider = resolveConfiguredProvider();
const providerBaseUrl = resolveProviderBaseUrl(provider);
emitDebug(options, `provider=${provider}`);
if (providerBaseUrl) {
  emitDebug(options, `provider.baseUrl=${JSON.stringify(providerBaseUrl)}`);
}
ensureProviderKey(provider);
```

with:

```ts
const provider = resolveConfiguredProvider();
emitDebug(options, `provider=${provider}`);

if (isAgentCliProvider(provider)) {
  const agentCliModelId = resolveModelId(options, provider);

  emitDebug(options, `model=${agentCliModelId}`);

  return runAgentCliRun(command, cwd, options, provider, agentCliModelId);
}

const providerBaseUrl = resolveProviderBaseUrl(provider);
if (providerBaseUrl) {
  emitDebug(options, `provider.baseUrl=${JSON.stringify(providerBaseUrl)}`);
}
ensureProviderKey(provider);
```

(The update no-op check already ran before this point, so agent-cli runs inherit it. The OpenRouter debug-fetch wrapper is installed after this branch, so it stays API-only.)

Add these two functions after `createRunUserMessage` (around line 321):

```ts
async function runAgentCliRun(
  command: OpenWikiCommand,
  cwd: string,
  options: OpenWikiRunOptions,
  provider: OpenWikiProvider,
  modelId: string,
): Promise<OpenWikiRunResult> {
  const context = await createRunContext(command, cwd);
  emitDebug(options, "context=created");
  const openWikiSnapshotBefore =
    command === "chat" ? null : await createOpenWikiContentSnapshot(cwd);
  emitDebug(options, "openwiki.snapshot=created");
  const threadId = options.threadId ?? createThreadId(cwd, createRunThreadId());
  emitDebug(options, `thread=${threadId}`);
  const resumeSessionId =
    options.isFollowup === true ? getThreadSessionId(threadId) : undefined;

  if (resumeSessionId) {
    emitDebug(options, `engine.resume session=${resumeSessionId}`);
  }

  const spec: EngineRunSpec = {
    command,
    cwd,
    modelId,
    prompt: createAgentCliRunUserMessage(command, cwd, context, options),
    systemPrompt: createSystemPrompt(command, "agent-cli"),
    resumeSessionId,
  };

  const outcome = await runAgentCli(
    getAgentCliAdapter(provider),
    getAgentCliProviderConfig(provider),
    spec,
    options,
  );

  if (outcome.sessionId) {
    setThreadSessionId(threadId, outcome.sessionId);
  }

  if (
    command !== "chat" &&
    openWikiSnapshotBefore !== (await createOpenWikiContentSnapshot(cwd))
  ) {
    await writeLastUpdateMetadata(command, cwd, modelId);
    emitDebug(options, "metadata=written");
  } else {
    emitDebug(
      options,
      command === "chat"
        ? "metadata=skipped command=chat"
        : "metadata=skipped openwiki=unchanged",
    );
  }

  return {
    command,
    model: modelId,
  };
}

function createAgentCliRunUserMessage(
  command: OpenWikiCommand,
  cwd: string,
  context: Awaited<ReturnType<typeof createRunContext>>,
  options: OpenWikiRunOptions,
): string {
  if (options.isFollowup === true && options.userMessage?.trim()) {
    return options.userMessage.trim();
  }

  return `
${createUserPrompt(command, context, options.userMessage ?? null)}

Repository root:
${cwd}

Runtime note:
- Treat the repository root above as the only project you are documenting.
- Your working directory is the repository root. Use repository-relative paths such as README.md and openwiki/quickstart.md with your file tools.
- Do not read or modify files outside this repository.
`.trim();
}
```

`OpenWikiProvider` is already imported as a type in this file; `createSystemPrompt`, `createUserPrompt`, `createRunContext`, `createOpenWikiContentSnapshot`, `writeLastUpdateMetadata`, `createThreadId`, and `createRunThreadId` are all already in scope.

- [ ] **Step 4: Run tests, lint, and build to verify everything passes**

Run: `pnpm vitest run test/agent-cli-run.test.ts` — Expected: PASS (2 tests).
Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/index.ts test/agent-cli-run.test.ts
git commit -m "feat: dispatch documentation runs to agent CLI engines"
```

---

### Task 8: Onboarding — setup-flow module, agent-check step, selectable provider

**Files:**

- Create: `src/credentials-flow.ts` (pure step logic extracted from `credentials.tsx` + agent-cli branches)
- Modify: `src/credentials.tsx` (use the flow module; add the `agent-check` step UI; hide key/LangSmith steps for agent-cli)
- Modify: `src/constants.ts:48-55` (add `"claude-code"` to `SELECTABLE_OPENWIKI_PROVIDERS`)
- Test: `test/credentials-flow.test.ts`

**Interfaces:**

- Consumes: `isAgentCliProvider`, `getAgentCliProviderConfig` (Task 1); `getAgentCliAdapter` (Task 5).
- Produces:
  - `src/credentials-flow.ts` exports: `type PromptStep = "agent-check" | "api-key" | "base-url" | "langsmith" | "model" | "provider"`, `needsCredentialSetup(modelIdOverride?)`, `needsBaseUrlStep(provider)`, `isBaseUrlConfigured(provider)`, `getInitialStep(modelIdOverride, provider)`, `getNextStepAfterProvider(provider, modelIdOverride)`, `getNextStepAfterAgentCheck(provider, modelIdOverride)`, `getNextStepAfterApiKey(provider, modelIdOverride)`, `getNextStepAfterBaseUrl(provider, modelIdOverride)`, `getNextStepAfterModel(provider)`.
  - `src/credentials.tsx` re-exports `needsCredentialSetup` so `src/cli.tsx`'s existing `import { needsCredentialSetup } from "./credentials.js"` keeps working unchanged.

- [ ] **Step 1: Write the failing tests**

Create `test/credentials-flow.test.ts`:

```ts
import { afterEach, beforeEach, describe, expect, test } from "vitest";
import {
  getInitialStep,
  getNextStepAfterAgentCheck,
  getNextStepAfterModel,
  getNextStepAfterProvider,
  needsCredentialSetup,
} from "../src/credentials-flow.ts";

const ENV_KEYS = [
  "OPENWIKI_PROVIDER",
  "OPENWIKI_MODEL_ID",
  "LANGSMITH_API_KEY",
  "ANTHROPIC_API_KEY",
  "OPENROUTER_API_KEY",
];
const savedEnv: Record<string, string | undefined> = {};

beforeEach(() => {
  for (const key of ENV_KEYS) {
    savedEnv[key] = process.env[key];
    delete process.env[key];
  }
});

afterEach(() => {
  for (const key of ENV_KEYS) {
    if (savedEnv[key] === undefined) delete process.env[key];
    else process.env[key] = savedEnv[key];
  }
});

describe("agent-cli setup flow", () => {
  test("claude-code needs setup only until provider and model are saved", () => {
    process.env.OPENWIKI_PROVIDER = "claude-code";
    expect(needsCredentialSetup(null)).toBe(true);

    process.env.OPENWIKI_MODEL_ID = "default";
    expect(needsCredentialSetup(null)).toBe(false);
  });

  test("api providers still require key and LangSmith decisions", () => {
    process.env.OPENWIKI_PROVIDER = "anthropic";
    process.env.OPENWIKI_MODEL_ID = "claude-sonnet-5";
    process.env.ANTHROPIC_API_KEY = "sk-test";
    expect(needsCredentialSetup(null)).toBe(true); // LangSmith undecided

    process.env.LANGSMITH_API_KEY = "";
    expect(needsCredentialSetup(null)).toBe(false);
  });

  test("provider selection routes claude-code to the agent check", () => {
    expect(getNextStepAfterProvider("claude-code", null)).toBe("agent-check");
    expect(getNextStepAfterProvider("anthropic", null)).toBe("api-key");
  });

  test("agent check advances to model selection when the model is unset", () => {
    expect(getNextStepAfterAgentCheck("claude-code", null)).toBe("model");
    expect(getNextStepAfterAgentCheck("claude-code", "opus")).toBe(null);
  });

  test("initial step for a configured claude-code without a model is agent-check", () => {
    process.env.OPENWIKI_PROVIDER = "claude-code";
    expect(getInitialStep(null, "claude-code")).toBe("agent-check");

    process.env.OPENWIKI_MODEL_ID = "default";
    expect(getInitialStep(null, "claude-code")).toBe(null);
  });

  test("model step skips LangSmith for agent-cli providers", () => {
    expect(getNextStepAfterModel("claude-code")).toBe(null);
    expect(getNextStepAfterModel("openrouter")).toBe("langsmith");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run test/credentials-flow.test.ts`
Expected: FAIL — cannot resolve `../src/credentials-flow.ts`.

- [ ] **Step 3: Create `src/credentials-flow.ts`**

```ts
import {
  getProviderApiKeyEnvKey,
  getProviderBaseUrlEnvKey,
  isAgentCliProvider,
  OPENWIKI_MODEL_ID_ENV_KEY,
  OPENWIKI_PROVIDER_ENV_KEY,
  providerRequiresBaseUrl,
  resolveConfiguredProvider,
  type OpenWikiProvider,
} from "./constants.js";

export type PromptStep =
  | "agent-check"
  | "api-key"
  | "base-url"
  | "langsmith"
  | "model"
  | "provider";

export function needsCredentialSetup(
  modelIdOverride: string | null = null,
): boolean {
  const provider = resolveConfiguredProvider();

  if (isAgentCliProvider(provider)) {
    return (
      process.env[OPENWIKI_PROVIDER_ENV_KEY] === undefined ||
      needsModelStep(modelIdOverride)
    );
  }

  return (
    process.env[OPENWIKI_PROVIDER_ENV_KEY] === undefined ||
    !process.env[getProviderApiKeyEnvKey(provider)] ||
    needsBaseUrlStep(provider) ||
    needsModelStep(modelIdOverride) ||
    process.env.LANGSMITH_API_KEY === undefined
  );
}

export function needsBaseUrlStep(provider: OpenWikiProvider): boolean {
  if (!providerRequiresBaseUrl(provider)) {
    return false;
  }

  return !isBaseUrlConfigured(provider);
}

export function isBaseUrlConfigured(provider: OpenWikiProvider): boolean {
  const baseUrlEnvKey = getProviderBaseUrlEnvKey(provider);

  return baseUrlEnvKey ? Boolean(process.env[baseUrlEnvKey]) : false;
}

function needsModelStep(modelIdOverride: string | null): boolean {
  return (
    modelIdOverride === null &&
    process.env[OPENWIKI_MODEL_ID_ENV_KEY] === undefined
  );
}

export function getInitialStep(
  modelIdOverride: string | null,
  provider: OpenWikiProvider,
): PromptStep | null {
  if (process.env[OPENWIKI_PROVIDER_ENV_KEY] === undefined) {
    return "provider";
  }

  if (isAgentCliProvider(provider)) {
    return needsModelStep(modelIdOverride) ? "agent-check" : null;
  }

  if (!process.env[getProviderApiKeyEnvKey(provider)]) {
    return "api-key";
  }

  if (needsBaseUrlStep(provider)) {
    return "base-url";
  }

  if (needsModelStep(modelIdOverride)) {
    return "model";
  }

  if (process.env.LANGSMITH_API_KEY === undefined) {
    return "langsmith";
  }

  return null;
}

export function getNextStepAfterProvider(
  provider: OpenWikiProvider,
  modelIdOverride: string | null,
): PromptStep | null {
  if (isAgentCliProvider(provider)) {
    return "agent-check";
  }

  if (!process.env[getProviderApiKeyEnvKey(provider)]) {
    return "api-key";
  }

  return getNextStepAfterApiKey(provider, modelIdOverride);
}

export function getNextStepAfterAgentCheck(
  provider: OpenWikiProvider,
  modelIdOverride: string | null,
): PromptStep | null {
  return needsModelStep(modelIdOverride) ? "model" : null;
}

export function getNextStepAfterApiKey(
  provider: OpenWikiProvider,
  modelIdOverride: string | null,
): PromptStep | null {
  if (needsBaseUrlStep(provider)) {
    return "base-url";
  }

  return getNextStepAfterBaseUrl(provider, modelIdOverride);
}

export function getNextStepAfterBaseUrl(
  provider: OpenWikiProvider,
  modelIdOverride: string | null,
): PromptStep | null {
  if (needsModelStep(modelIdOverride)) {
    return "model";
  }

  if (process.env.LANGSMITH_API_KEY === undefined) {
    return "langsmith";
  }

  return null;
}

export function getNextStepAfterModel(
  provider: OpenWikiProvider,
): PromptStep | null {
  if (isAgentCliProvider(provider)) {
    return null;
  }

  return process.env.LANGSMITH_API_KEY === undefined ? "langsmith" : null;
}
```

(The unused `provider` parameter on `getNextStepAfterAgentCheck` is intentional — it keeps all step functions uniform; prefix it with `_` if eslint complains: `_provider`.)

- [ ] **Step 4: Run the flow tests to verify they pass**

Run: `pnpm vitest run test/credentials-flow.test.ts`
Expected: PASS (6 tests).

- [ ] **Step 5: Rewire `src/credentials.tsx` to the flow module and add the agent-check step**

All edits below are in `src/credentials.tsx`:

1. Delete the local implementations of `needsCredentialSetup`, `needsBaseUrlStep`, `isBaseUrlConfigured`, `getInitialStep`, `getNextStepAfterProvider`, `getNextStepAfterApiKey`, `getNextStepAfterBaseUrl`, and the local `type PromptStep` declaration. Replace with:

```ts
import {
  getInitialStep,
  getNextStepAfterAgentCheck,
  getNextStepAfterApiKey,
  getNextStepAfterBaseUrl,
  getNextStepAfterModel,
  getNextStepAfterProvider,
  isBaseUrlConfigured,
  needsCredentialSetup,
  type PromptStep,
} from "./credentials-flow.js";
import { getAgentCliAdapter } from "./agent/engines/index.js";

export { needsCredentialSetup } from "./credentials-flow.js";
```

and add `getAgentCliProviderConfig, isAgentCliProvider,` to the `./constants.js` import list.

2. Add agent-check state inside `InitSetup` (next to the other `useState` calls):

```ts
type AgentCheckState =
  | { status: "checking" }
  | { status: "found"; version: string }
  | { status: "missing"; message: string };

const [agentCheck, setAgentCheck] = useState<AgentCheckState>({
  status: "checking",
});
const [agentCheckAttempt, setAgentCheckAttempt] = useState(0);
```

(Declare the `AgentCheckState` type at module level, below `PromptStep` usage, not inside the component.)

3. Add the check effect after the existing initial-step `useEffect`:

```ts
useEffect(() => {
  if (step !== "agent-check") {
    return;
  }

  let cancelled = false;

  setAgentCheck({ status: "checking" });

  void (async () => {
    const config = getAgentCliProviderConfig(provider);
    const binary =
      process.env[config.binaryEnvKey]?.trim() || config.defaultBinary;
    const status = await getAgentCliAdapter(provider).detectInstall(binary);

    if (cancelled) {
      return;
    }

    if (status.found) {
      setAgentCheck({ status: "found", version: status.version ?? "unknown" });
    } else {
      setAgentCheck({ status: "missing", message: config.installHint });
    }
  })();

  return () => {
    cancelled = true;
  };
}, [step, provider, agentCheckAttempt]);
```

4. In `useInput`, before the generic `if (key.return)` fallthrough, add:

```ts
if (step === "agent-check") {
  if (key.return && agentCheck.status === "found") {
    void submit();
  } else if (key.return && agentCheck.status === "missing") {
    setAgentCheckAttempt((attempt) => attempt + 1);
  }

  return;
}
```

5. In `submit()`, add a branch after the `step === "provider"` block:

```ts
if (step === "agent-check") {
  if (agentCheck.status !== "found") {
    return;
  }

  const nextStep = getNextStepAfterAgentCheck(provider, modelIdOverride);

  if (nextStep) {
    setIsCustomModelInput(false);
    setStep(nextStep);
    return;
  }

  await completeSetup({
    nextApiKey: null,
    nextBaseUrl: null,
    nextLangSmithKey: null,
    nextModelId: modelId,
    nextProvider: provider,
  });
  return;
}
```

6. In the `step === "model"` branch of `submit()`, replace:

```ts
if (process.env.LANGSMITH_API_KEY === undefined) {
  setStep("langsmith");
  return;
}
```

with:

```ts
const nextStep = getNextStepAfterModel(provider);

if (nextStep) {
  setStep(nextStep);
  return;
}
```

7. In the step-row rendering, replace the "Provider key" `<SetupStep .../>` with a kind-conditional pair, and hide LangSmith for agent-cli:

```tsx
        {isAgentCliProvider(provider) ? (
          <SetupStep
            label="Agent CLI"
            state={
              agentCheck.status === "found"
                ? "done"
                : step === "agent-check"
                  ? "current"
                  : "pending"
            }
            detail={
              agentCheck.status === "found"
                ? `found version ${agentCheck.version}`
                : "verify the subscription agent CLI is installed"
            }
          />
        ) : (
          <SetupStep
            label="Provider key"
            ...existing props unchanged...
          />
        )}
```

and wrap the LangSmith `<SetupStep .../>` in `{isAgentCliProvider(provider) ? null : ( ... )}`. (The Base URL row is already conditional on `providerRequiresBaseUrl`, which is `false` for agent-cli.)

8. Pass `agentCheck` into `Prompt` (add to `PromptProps` as `agentCheck: AgentCheckState`) and add to the `Prompt` component before the `api-key` branch:

```tsx
if (step === "agent-check") {
  if (agentCheck.status === "checking") {
    return <Text>Checking for the {getProviderLabel(provider)} CLI...</Text>;
  }

  if (agentCheck.status === "found") {
    return (
      <Box flexDirection="column">
        <Text>
          Found the {getProviderLabel(provider)} CLI{" "}
          <Text color="green">{agentCheck.version}</Text>.
        </Text>
        <Text color="gray">
          No API key needed — runs use your subscription login. Press Enter to
          continue.
        </Text>
      </Box>
    );
  }

  return (
    <Box flexDirection="column">
      <Text color="red">Agent CLI not found.</Text>
      <Text>{agentCheck.message}</Text>
      <Text color="gray">Press Enter to check again.</Text>
    </Box>
  );
}
```

9. In `getProviderArticle`, include the new provider in the "a" list: `return provider === "baseten" || provider === "fireworks" || provider === "claude-code" ? "a" : "an";`

10. In `src/constants.ts`, add `"claude-code",` to the end of `SELECTABLE_OPENWIKI_PROVIDERS`.

- [ ] **Step 6: Run tests, lint, and build to verify everything passes**

Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

- [ ] **Step 7: Smoke-test the onboarding UI manually**

Run in a scratch directory (NOT this repo, to avoid touching `~/.openwiki/.env` semantics for your real setup — it is fine locally since your provider is already configured; the setup UI only appears when config is missing):

```bash
cd "$(mktemp -d)" && git init -q . && echo hi > README.md
OPENWIKI_PROVIDER= OPENWIKI_MODEL_ID= node /path/to/openwiki/dist/cli.js --dry-run
```

Expected: the provider list includes "Claude Code (subscription) (claude-code)"; selecting it shows the Agent CLI check (found version, since Claude Code is installed on this machine), then model selection (`Subscription default` first), then completes without API-key or LangSmith prompts. Press Ctrl+C to exit without saving if you want to avoid writing `~/.openwiki/.env`; if you did save, restore your previous `OPENWIKI_PROVIDER`/`OPENWIKI_MODEL_ID` in `~/.openwiki/.env` afterwards.

- [ ] **Step 8: Commit**

```bash
git add src/credentials-flow.ts src/credentials.tsx src/constants.ts test/credentials-flow.test.ts
git commit -m "feat: onboard claude-code with an install check instead of an API key"
```

---

### Task 9: Documentation

**Files:**

- Modify: `README.md` (new subsection under "## Customizing")
- Modify: `openwiki/quickstart.md:9` and `openwiki/quickstart.md:49`

**Interfaces:** none (docs only).

- [ ] **Step 1: Add the README section**

In `README.md`, insert after the paragraph under `## Customizing` (the one ending "...specify your own custom model ID.") and before `### Alternative base URLs`:

````markdown
### Use your coding-agent subscription (no API key)

If you already pay for a subscription coding agent, OpenWiki can delegate
documentation runs to it instead of calling a metered API. The first supported
agent is Claude Code:

```bash
OPENWIKI_PROVIDER=claude-code
OPENWIKI_MODEL_ID=default   # or sonnet / opus / haiku
```

Requirements and notes:

- Install Claude Code (`npm install -g @anthropic-ai/claude-code`) and run
  `claude` once to complete the subscription login.
- No API key is stored. Runs execute through the Claude Code CLI in headless
  mode with a documentation-scoped tool allowlist, using your existing login.
- Set `OPENWIKI_CLAUDE_CODE_BINARY` to point at a non-default binary location,
  and `OPENWIKI_AGENT_CLI_TIMEOUT_SECONDS` to change the 30-minute run timeout.
- Local runs only for now: the scheduled GitHub Action still needs an API-key
  provider.
- LangSmith tracing does not apply to delegated runs.
````

- [ ] **Step 2: Update `openwiki/quickstart.md`**

Line 9 — replace:

```markdown
- Supports multiple model providers — OpenRouter (default), Anthropic, OpenAI, Baseten, and Fireworks — each with their own API key and model list.
```

with:

```markdown
- Supports multiple model providers — OpenRouter (default), Anthropic, OpenAI, Baseten, and Fireworks — each with their own API key and model list, plus subscription agent-CLI providers such as Claude Code that run without an API key.
```

Line 49 — replace:

```markdown
- Provider support is centralized in `src/constants.ts`. Adding or changing a provider means updating `PROVIDER_CONFIGS`, the `OpenWikiProvider` type, and the model-creation branch in `src/agent/index.ts`.
```

with:

```markdown
- Provider support is centralized in `src/constants.ts`. API providers (`kind: "api"`) add a `PROVIDER_CONFIGS` entry, the `OpenWikiProvider` type member, and a model-creation branch in `src/agent/index.ts`. Subscription agent-CLI providers (`kind: "agent-cli"`) add a config entry plus an adapter in `src/agent/engines/` registered in `src/agent/engines/index.ts`.
```

The remaining wiki pages (`architecture/overview.md`, `cli/usage.md`, `agent/workflow.md`, `operations/credentials-and-updates.md`) are generated documentation; they get refreshed by the dogfooding `openwiki --update` run in Task 10 rather than hand-edited here.

- [ ] **Step 3: Verify formatting and commit**

Run: `pnpm run format:check` — Expected: PASS (run `pnpm run format` first if it flags the new README block).

```bash
git add README.md openwiki/quickstart.md
git commit -m "docs: document the claude-code subscription provider"
```

---

### Task 10: Live end-to-end verification with real Claude Code

This task needs the real `claude` CLI logged in on the machine (it is, on this one). It consumes a small amount of subscription quota.

**Files:**

- Possibly modify: `src/agent/engines/claude-code.ts` + `test/claude-code-adapter.test.ts` (only if the live event format differs from Task 4's assumptions)
- Possibly modify: `openwiki/*.md` (via the dogfooding update run)

**Interfaces:** none new.

- [ ] **Step 1: Validate the CLI flags against the installed version**

```bash
claude --help 2>&1 | grep -E "output-format|append-system-prompt|allowedTools|permission-mode|resume"
```

Expected: all five flags listed. If a flag is missing or renamed in the installed version, update `claudeCodeAdapter.buildArgs` and its tests to the supported spelling before continuing.

- [ ] **Step 2: Capture a real stream-json sample and check parser assumptions**

```bash
echo "Reply with the single word ok" | claude -p --output-format stream-json --verbose --max-turns 1 > /tmp/claude-stream-sample.ndjson
head -c 2000 /tmp/claude-stream-sample.ndjson
```

Expected: first line is a `{"type":"system","subtype":"init","session_id":...}` object; an `assistant` line with `content` blocks; a final `{"type":"result","subtype":"success",...}` line. If field names differ from Task 4's fixtures, update `parseEvent` and the fixtures in `test/claude-code-adapter.test.ts` to match reality, and re-run `pnpm test`.

- [ ] **Step 3: Run a real init on a small fixture repo**

```bash
pnpm run build
FIXTURE=$(mktemp -d) && cd "$FIXTURE" && git init -q . \
  && printf '# Tiny\n\nA tiny fixture.\n' > README.md \
  && printf 'export const add = (a, b) => a + b;\n' > index.js \
  && git add . && git commit -qm initial
OPENWIKI_PROVIDER=claude-code OPENWIKI_MODEL_ID=default \
  node /path/to/openwiki/dist/cli.js --print --init
```

Expected: streamed text/tool events render in the terminal; on completion `openwiki/quickstart.md` and `openwiki/.last-update.json` exist (`model` field is `default`, `command` is `init`), and `AGENTS.md` was created with the OpenWiki reference section. Verify no `~/.openwiki/.env` key was required (unset `ANTHROPIC_API_KEY` etc. in the invocation environment to prove it: prefix the command with `env -u ANTHROPIC_API_KEY -u OPENROUTER_API_KEY`).

- [ ] **Step 4: Verify interactive follow-up resumes the session**

From the same fixture repo, run `OPENWIKI_PROVIDER=claude-code node /path/to/openwiki/dist/cli.js`, send "What did you document?", then a follow-up "Summarize it in one sentence." With `--debug`-less UI this is behavioral only: the second reply should show awareness of the first exchange (session resumed). If it clearly does not, check `getThreadSessionId`/`setThreadSessionId` wiring in `runAgentCliRun`.

- [ ] **Step 5: Dogfood on this repository**

From the OpenWiki repo root on the feature branch:

```bash
OPENWIKI_PROVIDER=claude-code OPENWIKI_MODEL_ID=default node dist/cli.js --print --update
```

Expected: an update run that refreshes the wiki pages affected by this feature (architecture/CLI/credentials pages). Review `git diff openwiki/` for accuracy — the update should mention the agent-cli provider kind and engines module. Discard low-value churn per the repo's surgical-update rules.

- [ ] **Step 6: Full suite, then commit any adjustments**

Run: `pnpm test && pnpm run lint:check && pnpm run build` — Expected: all green.

```bash
git add -A
git commit -m "test: verify claude-code engine against the live CLI and refresh wiki"
```

---

## Self-Review (completed during planning)

- **Spec coverage:** provider union + config (Task 1); engines module with adapter/runner/registry, event mapping, timeout, stderr tail, session resume (Tasks 3-5); runtime branch + engine-aware prompts + runtime notes (Tasks 6-7); onboarding with install check, no secrets, LangSmith skip, diagnostics (Tasks 1, 8); errors (Task 5 `formatRunFailure` with login hint); docs (Task 9); manual verification milestone incl. follow-up resume (Task 10). CI tokens, Codex adapter: explicitly out of scope per spec.
- **Placeholder scan:** the only intentionally deferred content is live-format validation (Task 10 Steps 1-2), which the spec itself flags as implementation-time verification; every code step has complete code.
- **Type consistency:** `EngineRunSpec`/`AgentCliEvent`/`AgentCliAdapter` (Task 3) match usage in Tasks 4, 5, 7; `AgentCliProviderConfig` fields (Task 1) match `runner.ts` and `credentials.tsx` usage; `PromptStep` moves to `credentials-flow.ts` with the same member spelling; `getNextStepAfterModel` replaces the inline LangSmith check in both flow and UI.
