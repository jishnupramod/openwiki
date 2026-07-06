# IBM Bob (Bob Shell) Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `ibm-bob` as the second `agent-cli` adapter so OpenWiki can delegate documentation runs to a locally installed, subscription-authenticated IBM Bob Shell (`bob`) CLI.

**Architecture:** One new adapter module (`src/agent/engines/ibm-bob.ts`) behind the existing `AgentCliAdapter` interface, plus a provider entry and env-key registrations. Bob has no system-prompt flag, so the interface gains one optional hook, `buildStdin`, letting an adapter compose the stdin payload (the runner defaults to `spec.prompt`). Everything else — dispatch, credentials flow, prompt engine, runner process management — is already config-driven and unchanged.

**Tech Stack:** TypeScript ESM (`"type": "module"`, `.js` suffixes on relative imports in `src/`, tests import `../src/<file>.ts`), vitest, pnpm, Node ≥ 20. Zero new npm dependencies.

## Global Constraints

- Provider id is exactly `ibm-bob`; UI label is exactly `IBM Bob (subscription)`.
- Binary override env key is exactly `OPENWIKI_IBM_BOB_BINARY` (exported const `IBM_BOB_BINARY_ENV_KEY`); default binary is exactly `bob`.
- Model options are exactly `[{ id: "default", label: "Subscription default" }]` — no named models. Any non-`"default"` model id passes through as `--model <id>`.
- Bob invocation flags (order matters for the buildArgs test): `--output-format stream-json --approval-mode auto_edit --chat-mode advanced --allowed-tools <IBM_BOB_ALLOWED_TOOLS>`, then optional `--model <id>`, then optional `--resume <sessionId>`. No positional prompt and no `-p` — the prompt rides stdin.
- The system prompt is delivered by prepending it to the stdin payload: `` `${spec.systemPrompt}\n\n${spec.prompt}` `` (Bob Shell has no `--append-system-prompt` equivalent).
- Bob stream-json event schema (verified against the installed v1.0.3 bundle — fixtures must match these shapes exactly): `{type:"init", timestamp, session_id, model}`; `{type:"message", timestamp, role:"user"|"assistant", content, delta?}`; `{type:"tool_use", timestamp, tool_name, tool_id, parameters}`; `{type:"tool_result", timestamp, tool_id, status:"success"|"error", output, error?:{type,message}}`; `{type:"error", timestamp, severity:"warning"|"error", message}`; `{type:"result", timestamp, status:"success"|"error", error?:{type,message}, stats}`.
- No new npm dependencies. No changes to `src/agent/index.ts`, `src/cli.tsx`, `src/credentials-flow.ts`, `src/credentials.tsx`, or `src/agent/prompt.ts`.
- All existing tests must keep passing (`pnpm test`); also run `pnpm run lint` and `pnpm run build` before each commit.
- Commits are authored by `jishnupramod <jishnumagnanimous@gmail.com>` (repo-local git config already set) and end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: `ibm-bob` provider entry and env-key registration

**Files:**
- Modify: `src/constants.ts` (env const near line 11, `OpenWikiProvider` union at lines 17–24, `SELECTABLE_OPENWIKI_PROVIDERS` at lines 64–72, `PROVIDER_CONFIGS` after the `fireworks` entry ~line 111)
- Modify: `src/env.ts` (import block lines 4–18, `managedEnvKeys` lines 37–52, `getCredentialDiagnostics` lines 81–94, `isNonSecretDiagnosticKey` lines 179–187)
- Test: `test/provider-kinds.test.ts`

**Interfaces:**
- Consumes: existing helpers `normalizeProvider`, `isAgentCliProvider`, `getAgentCliProviderConfig`, `getProviderModelOptions`, `getDefaultModelId`, `getProviderLabel`, `getProviderApiKeyEnvKey`, `formatProviderSwitchNotice` (all in `src/constants.ts`, all kind-driven — no changes to them).
- Produces: `IBM_BOB_BINARY_ENV_KEY` (exported const, value `"OPENWIKI_IBM_BOB_BINARY"`), `"ibm-bob"` as a valid `OpenWikiProvider`, and `PROVIDER_CONFIGS["ibm-bob"]` with `kind: "agent-cli"`, `defaultBinary: "bob"`. Tasks 3–5 rely on these exact names.

- [ ] **Step 1: Write the failing tests**

Append to `test/provider-kinds.test.ts` (add `IBM_BOB_BINARY_ENV_KEY` and `getProviderModelOptions` to the existing import from `../src/constants.ts`):

```ts
describe("ibm-bob provider entry", () => {
  test("ibm-bob is a valid, selectable provider id", () => {
    expect(normalizeProvider("ibm-bob")).toBe("ibm-bob");
    expect(normalizeProvider("IBM-BOB")).toBe("ibm-bob");
    expect(isAgentCliProvider("ibm-bob")).toBe(true);
    expect(SELECTABLE_OPENWIKI_PROVIDERS).toContain("ibm-bob");
  });

  test("agent-cli config exposes the bob binary, override key, and install hint", () => {
    const config = getAgentCliProviderConfig("ibm-bob");

    expect(config.kind).toBe("agent-cli");
    expect(config.defaultBinary).toBe("bob");
    expect(config.binaryEnvKey).toBe(IBM_BOB_BINARY_ENV_KEY);
    expect(IBM_BOB_BINARY_ENV_KEY).toBe("OPENWIKI_IBM_BOB_BINARY");
    expect(config.installHint).toContain("bob");
    expect(config.installHint).toContain("trust");
  });

  test("only the subscription default model is offered", () => {
    expect(getDefaultModelId("ibm-bob")).toBe("default");
    expect(getProviderModelOptions("ibm-bob")).toEqual([
      { id: "default", label: "Subscription default" },
    ]);
  });

  test("label reads as a subscription provider", () => {
    expect(getProviderLabel("ibm-bob")).toBe("IBM Bob (subscription)");
  });

  test("api-key helper rejects ibm-bob", () => {
    expect(() => getProviderApiKeyEnvKey("ibm-bob")).toThrow(/ibm-bob/);
  });

  test("switch notice mentions the CLI login instead of a key", () => {
    const notice = formatProviderSwitchNotice("ibm-bob");

    expect(notice).toContain("Provider switched to IBM Bob (subscription)");
    expect(notice).not.toContain("_API_KEY");
    expect(notice).toContain("login");
  });

  test("credential diagnostics include the ibm-bob binary override", async () => {
    const diagnostics = await getCredentialDiagnostics();

    expect(diagnostics.map((diagnostic) => diagnostic.key)).toContain(
      IBM_BOB_BINARY_ENV_KEY,
    );
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/provider-kinds.test.ts`
Expected: FAIL — TypeScript/ESM error that `IBM_BOB_BINARY_ENV_KEY` is not exported from `../src/constants.ts`.

- [ ] **Step 3: Implement the provider entry**

In `src/constants.ts`:

(a) After line 11 (`export const CLAUDE_CODE_BINARY_ENV_KEY = ...`), add:

```ts
export const IBM_BOB_BINARY_ENV_KEY = "OPENWIKI_IBM_BOB_BINARY";
```

(b) In the `OpenWikiProvider` union, add `| "ibm-bob"` after `| "fireworks"`:

```ts
export type OpenWikiProvider =
  | "anthropic"
  | "baseten"
  | "claude-code"
  | "fireworks"
  | "ibm-bob"
  | "openai"
  | "openai-compatible"
  | "openrouter";
```

(c) In `SELECTABLE_OPENWIKI_PROVIDERS`, append `"ibm-bob"` after `"claude-code"` (the agent-cli providers group at the end of the menu):

```ts
export const SELECTABLE_OPENWIKI_PROVIDERS = [
  "openrouter",
  "baseten",
  "fireworks",
  "openai",
  "openai-compatible",
  "anthropic",
  "claude-code",
  "ibm-bob",
] as const satisfies readonly SelectableOpenWikiProvider[];
```

(d) In `PROVIDER_CONFIGS`, insert after the `fireworks` entry (keys are alphabetical):

```ts
  "ibm-bob": {
    kind: "agent-cli",
    binaryEnvKey: IBM_BOB_BINARY_ENV_KEY,
    defaultBinary: "bob",
    installHint:
      "Install Bob Shell (curl -fsSL https://bob.ibm.com/download/bobshell.sh | bash), run `bob` once in this repository to complete the IBMid login, and trust the folder when prompted.",
    label: "IBM Bob (subscription)",
    modelOptions: [{ id: "default", label: "Subscription default" }],
  },
```

In `src/env.ts`:

(e) Add `IBM_BOB_BINARY_ENV_KEY,` to the import block from `./constants.js` (alphabetical: after `FIREWORKS_API_KEY_ENV_KEY,`).

(f) In `managedEnvKeys`, add `IBM_BOB_BINARY_ENV_KEY,` on the line after `CLAUDE_CODE_BINARY_ENV_KEY,`.

(g) In `getCredentialDiagnostics`, add after the `CLAUDE_CODE_BINARY_ENV_KEY` line:

```ts
    createCredentialDiagnostic(IBM_BOB_BINARY_ENV_KEY, fileEnv),
```

(h) In `isNonSecretDiagnosticKey`, add `key === IBM_BOB_BINARY_ENV_KEY` as a new `||` arm after the `CLAUDE_CODE_BINARY_ENV_KEY` arm (without it, a configured binary path renders masked like a secret):

```ts
function isNonSecretDiagnosticKey(key: string): boolean {
  return (
    key === OPENWIKI_MODEL_ID_ENV_KEY ||
    key === OPENWIKI_PROVIDER_ENV_KEY ||
    key === ANTHROPIC_BASE_URL_ENV_KEY ||
    key === OPENAI_COMPATIBLE_BASE_URL_ENV_KEY ||
    key === CLAUDE_CODE_BINARY_ENV_KEY ||
    key === IBM_BOB_BINARY_ENV_KEY
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run test/provider-kinds.test.ts`
Expected: PASS (all existing + 7 new tests).

Run: `pnpm test && pnpm run lint && pnpm run build`
Expected: everything green — the union widening must not break any exhaustive `Record<OpenWikiProvider, …>` sites other than `PROVIDER_CONFIGS` (there are none).

- [ ] **Step 5: Commit**

```bash
git add src/constants.ts src/env.ts test/provider-kinds.test.ts
git commit -m "feat: add ibm-bob provider entry and binary env key

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: `buildStdin` adapter hook and runner support

**Files:**
- Modify: `src/agent/engines/types.ts` (the `AgentCliAdapter` type, lines 26–32)
- Modify: `src/agent/engines/runner.ts` (line 152, the `child.stdin.write(spec.prompt)` call)
- Test: `test/agent-cli-runner.test.ts`

**Interfaces:**
- Consumes: `EngineRunSpec`, `AgentCliAdapter` from `src/agent/engines/types.ts`; the existing `SUCCESS_STUB` in the test file (its text event echoes `"prompt-bytes:" + input.length`, so stdin size is observable).
- Produces: `AgentCliAdapter.buildStdin?(spec: EngineRunSpec): string` — optional; the runner sends `adapter.buildStdin?.(spec) ?? spec.prompt` on stdin. `AgentCliAdapter.id` becomes `"claude-code" | "ibm-bob"`. Task 3's adapter implements both.

- [ ] **Step 1: Write the failing test**

In `test/agent-cli-runner.test.ts`, add inside the existing `describe("runAgentCli", ...)` block:

```ts
  test("sends the adapter-composed stdin payload when buildStdin is present", async () => {
    process.env[CLAUDE_CODE_BINARY_ENV_KEY] = await writeStub(
      stubDir,
      "stub-stdin",
      SUCCESS_STUB,
    );
    const events: OpenWikiRunEvent[] = [];
    const adapterWithStdin = {
      ...claudeCodeAdapter,
      buildStdin: (spec: EngineRunSpec) =>
        `${spec.systemPrompt}\n\n${spec.prompt}`,
    };

    await runAgentCli(
      adapterWithStdin,
      getAgentCliProviderConfig("claude-code"),
      baseSpec,
      { onEvent: (event) => events.push(event) },
    );

    const expectedBytes = `${baseSpec.systemPrompt}\n\n${baseSpec.prompt}`
      .length;
    const text = events.find((event) => event.type === "text");
    expect(text).toMatchObject({ text: `prompt-bytes:${expectedBytes}` });
  });
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run test/agent-cli-runner.test.ts -t "adapter-composed stdin"`
Expected: FAIL — TypeScript rejects `buildStdin` (not a property of `AgentCliAdapter`), or at runtime the text event reports `prompt-bytes:${baseSpec.prompt.length}` (16) instead of the combined length (34).

- [ ] **Step 3: Implement the hook**

In `src/agent/engines/types.ts`, replace the `AgentCliAdapter` type with:

```ts
export type AgentCliAdapter = {
  id: "claude-code" | "ibm-bob";
  detectInstall(binary: string): Promise<AgentCliInstallStatus>;
  buildArgs(spec: EngineRunSpec): string[];
  /**
   * Composes the full stdin payload for vendors without a system-prompt
   * flag. When absent, the runner sends spec.prompt unchanged.
   */
  buildStdin?(spec: EngineRunSpec): string;
  /** Parses one NDJSON line of vendor output; unknown lines return []. */
  parseEvent(line: unknown): AgentCliEvent[];
};
```

In `src/agent/engines/runner.ts`, change line 152 from `child.stdin.write(spec.prompt);` to:

```ts
  child.stdin.write(adapter.buildStdin?.(spec) ?? spec.prompt);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run test/agent-cli-runner.test.ts`
Expected: PASS — the new test and all existing runner tests (the claude adapter has no `buildStdin`, so existing behavior is unchanged).

Run: `pnpm test && pnpm run lint && pnpm run build`
Expected: green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/engines/types.ts src/agent/engines/runner.ts test/agent-cli-runner.test.ts
git commit -m "feat: let agent-cli adapters compose the stdin payload

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: The `ibm-bob` adapter

**Files:**
- Create: `src/agent/engines/ibm-bob.ts`
- Test: `test/ibm-bob-adapter.test.ts` (new)

**Interfaces:**
- Consumes: `AgentCliAdapter`, `AgentCliEvent`, `AgentCliInstallStatus`, `EngineRunSpec` from `./types.js` (Task 2 shape); `formatToolArgs` from `../tool-format.js` (existing: renders a tool-args record as `key="value"` pairs).
- Produces: `ibmBobAdapter: AgentCliAdapter` (id `"ibm-bob"`) and `IBM_BOB_ALLOWED_TOOLS: string`. Task 4 registers `ibmBobAdapter` and imports both in tests.

- [ ] **Step 1: Write the failing tests**

Create `test/ibm-bob-adapter.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import {
  IBM_BOB_ALLOWED_TOOLS,
  ibmBobAdapter,
} from "../src/agent/engines/ibm-bob.ts";
import type { EngineRunSpec } from "../src/agent/engines/types.ts";

const baseSpec: EngineRunSpec = {
  command: "init",
  cwd: "/tmp/repo",
  modelId: "default",
  prompt: "Initialize docs.",
  systemPrompt: "You are OpenWiki.",
};

describe("ibmBobAdapter.buildArgs", () => {
  test("builds headless stream-json args with pinned approval and chat modes", () => {
    expect(ibmBobAdapter.buildArgs(baseSpec)).toEqual([
      "--output-format",
      "stream-json",
      "--approval-mode",
      "auto_edit",
      "--chat-mode",
      "advanced",
      "--allowed-tools",
      IBM_BOB_ALLOWED_TOOLS,
    ]);
  });

  test("omits --model for the subscription default and adds it otherwise", () => {
    expect(ibmBobAdapter.buildArgs(baseSpec)).not.toContain("--model");

    const args = ibmBobAdapter.buildArgs({
      ...baseSpec,
      modelId: "granite-3-3-8b-instruct",
    });

    expect(args[args.indexOf("--model") + 1]).toBe("granite-3-3-8b-instruct");
  });

  test("adds --resume for follow-up sessions", () => {
    const args = ibmBobAdapter.buildArgs({
      ...baseSpec,
      resumeSessionId: "bob-sess-1",
    });

    expect(args[args.indexOf("--resume") + 1]).toBe("bob-sess-1");
  });

  test("never passes the prompt as an argument", () => {
    const args = ibmBobAdapter.buildArgs(baseSpec);

    expect(args).not.toContain("-p");
    expect(args).not.toContain(baseSpec.prompt);
    expect(args).not.toContain(baseSpec.systemPrompt);
  });

  test("allowed tools stay documentation-shaped", () => {
    const tools = IBM_BOB_ALLOWED_TOOLS.split(",");

    expect(tools).toContain("run_shell_command(git log)");
    expect(tools).toContain("run_shell_command(rm -f openwiki/_plan.md)");
    expect(IBM_BOB_ALLOWED_TOOLS).not.toContain("web_fetch");
    expect(IBM_BOB_ALLOWED_TOOLS).not.toContain("google_web_search");
  });
});

describe("ibmBobAdapter.buildStdin", () => {
  test("prepends the system prompt to the user prompt", () => {
    expect(ibmBobAdapter.buildStdin?.(baseSpec)).toBe(
      "You are OpenWiki.\n\nInitialize docs.",
    );
  });
});

describe("ibmBobAdapter.detectInstall", () => {
  test("reports a missing binary", async () => {
    const status = await ibmBobAdapter.detectInstall(
      "definitely-not-a-real-binary-xyz",
    );

    expect(status.found).toBe(false);
  });

  test("reports a version for an executable that prints one", async () => {
    const status = await ibmBobAdapter.detectInstall(process.execPath);

    expect(status.found).toBe(true);
    expect(status.version).toMatch(/\d+\.\d+/);
  });
});

describe("ibmBobAdapter.parseEvent", () => {
  test("init yields a session event and a debug event", () => {
    const events = ibmBobAdapter.parseEvent({
      type: "init",
      timestamp: "2026-07-06T00:00:00.000Z",
      session_id: "3f6b1a2c-0000-0000-0000-000000000000",
      model: "bob-default",
    });

    expect(events).toEqual([
      {
        type: "session",
        sessionId: "3f6b1a2c-0000-0000-0000-000000000000",
      },
      {
        type: "openwiki",
        event: {
          type: "debug",
          message: "ibm-bob session initialized model=bob-default",
        },
      },
    ]);
  });

  test("assistant message deltas become text events", () => {
    const events = ibmBobAdapter.parseEvent({
      type: "message",
      timestamp: "2026-07-06T00:00:00.000Z",
      role: "assistant",
      content: "Working on it.",
      delta: true,
    });

    expect(events).toEqual([
      {
        type: "openwiki",
        event: { source: "main", type: "text", text: "Working on it." },
      },
    ]);
  });

  test("user message echoes are ignored", () => {
    expect(
      ibmBobAdapter.parseEvent({
        type: "message",
        timestamp: "2026-07-06T00:00:00.000Z",
        role: "user",
        content: "Initialize docs.",
      }),
    ).toEqual([]);
  });

  test("tool_use becomes a tool_start event", () => {
    const events = ibmBobAdapter.parseEvent({
      type: "tool_use",
      timestamp: "2026-07-06T00:00:00.000Z",
      tool_name: "write_to_file",
      tool_id: "bob-tool-1",
      parameters: { path: "openwiki/quickstart.md" },
    });

    expect(events).toEqual([
      {
        type: "openwiki",
        event: {
          type: "tool_start",
          call: 'write_to_file(path="openwiki/quickstart.md")',
          id: "bob-tool-1",
          input: { path: "openwiki/quickstart.md" },
          name: "write_to_file",
        },
      },
    ]);
  });

  test("tool_result becomes a tool_end event with status mapping", () => {
    const ok = ibmBobAdapter.parseEvent({
      type: "tool_result",
      timestamp: "2026-07-06T00:00:00.000Z",
      tool_id: "bob-tool-1",
      status: "success",
      output: "written",
    });
    const failed = ibmBobAdapter.parseEvent({
      type: "tool_result",
      timestamp: "2026-07-06T00:00:00.000Z",
      tool_id: "bob-tool-2",
      status: "error",
      output: "denied",
      error: { type: "TOOL_EXECUTION_ERROR", message: "denied" },
    });

    expect(ok).toEqual([
      {
        type: "openwiki",
        event: {
          type: "tool_end",
          id: "bob-tool-1",
          name: "tool",
          status: "finished",
        },
      },
    ]);
    expect(failed).toEqual([
      {
        type: "openwiki",
        event: {
          type: "tool_end",
          id: "bob-tool-2",
          name: "tool",
          status: "error",
        },
      },
    ]);
  });

  test("error events become debug events", () => {
    expect(
      ibmBobAdapter.parseEvent({
        type: "error",
        timestamp: "2026-07-06T00:00:00.000Z",
        severity: "warning",
        message: "Loop detected, stopping execution",
      }),
    ).toEqual([
      {
        type: "openwiki",
        event: {
          type: "debug",
          message: "ibm-bob warning: Loop detected, stopping execution",
        },
      },
    ]);
  });

  test("result events map success and error statuses", () => {
    expect(
      ibmBobAdapter.parseEvent({
        type: "result",
        timestamp: "2026-07-06T00:00:00.000Z",
        status: "success",
        stats: {},
      }),
    ).toEqual([{ type: "result", ok: true, errorMessage: undefined }]);

    expect(
      ibmBobAdapter.parseEvent({
        type: "result",
        timestamp: "2026-07-06T00:00:00.000Z",
        status: "error",
        error: {
          type: "FatalToolExecutionError",
          message: "Authentication timeout (3 minutes)",
        },
        stats: {},
      }),
    ).toEqual([
      {
        type: "result",
        ok: false,
        errorMessage: "Authentication timeout (3 minutes)",
      },
    ]);
  });

  test("a result error without a message gets a readable fallback", () => {
    expect(
      ibmBobAdapter.parseEvent({
        type: "result",
        timestamp: "2026-07-06T00:00:00.000Z",
        status: "error",
        stats: {},
      }),
    ).toEqual([
      {
        type: "result",
        ok: false,
        errorMessage: "IBM Bob run ended with error.",
      },
    ]);
  });

  test("unknown lines are ignored", () => {
    expect(ibmBobAdapter.parseEvent("not json-shaped")).toEqual([]);
    expect(ibmBobAdapter.parseEvent({ type: "mystery" })).toEqual([]);
    expect(ibmBobAdapter.parseEvent(null)).toEqual([]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/ibm-bob-adapter.test.ts`
Expected: FAIL — cannot resolve `../src/agent/engines/ibm-bob.ts`.

- [ ] **Step 3: Implement the adapter**

Create `src/agent/engines/ibm-bob.ts`:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";
import { formatToolArgs } from "../tool-format.js";
import type {
  AgentCliAdapter,
  AgentCliEvent,
  AgentCliInstallStatus,
  EngineRunSpec,
} from "./types.js";

const execFileAsync = promisify(execFile);

/**
 * Documentation-shaped shell allowlist: read-only git plus the single exact
 * rm needed to clean up the temporary plan file. Bob Shell's file read/edit
 * tools are auto-approved by --approval-mode auto_edit rather than listed
 * here; network tools stay unapproved on purpose (headless runs cannot answer
 * their confirmation prompts). Bob confines writes to the directory it was
 * started in; the runner spawns the CLI with cwd set to the repository root
 * and never passes --include-directories, so that boundary is exactly the
 * target repository. Bob refuses non-default approval modes in untrusted
 * folders, so the repository must be trusted in Bob (run `bob` there once).
 */
export const IBM_BOB_ALLOWED_TOOLS = [
  "run_shell_command(git log)",
  "run_shell_command(git show)",
  "run_shell_command(git diff)",
  "run_shell_command(git status)",
  "run_shell_command(git blame)",
  "run_shell_command(git rev-parse)",
  "run_shell_command(rm -f openwiki/_plan.md)",
].join(",");

export const ibmBobAdapter: AgentCliAdapter = {
  id: "ibm-bob",

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
      "--output-format",
      "stream-json",
      "--approval-mode",
      "auto_edit",
      "--chat-mode",
      "advanced",
      "--allowed-tools",
      IBM_BOB_ALLOWED_TOOLS,
    ];

    if (spec.modelId !== "default") {
      args.push("--model", spec.modelId);
    }

    if (spec.resumeSessionId) {
      args.push("--resume", spec.resumeSessionId);
    }

    return args;
  },

  buildStdin(spec: EngineRunSpec): string {
    // Bob Shell has no --append-system-prompt equivalent, so the system
    // prompt travels as a preamble of the stdin payload.
    return `${spec.systemPrompt}\n\n${spec.prompt}`;
  },

  parseEvent(line: unknown): AgentCliEvent[] {
    if (!isRecord(line) || typeof line.type !== "string") {
      return [];
    }

    if (line.type === "init") {
      return parseInitEvent(line);
    }

    if (line.type === "message") {
      return parseMessageEvent(line);
    }

    if (
      line.type === "tool_use" &&
      typeof line.tool_id === "string" &&
      typeof line.tool_name === "string"
    ) {
      return [
        {
          type: "openwiki",
          event: {
            type: "tool_start",
            call: `${line.tool_name}(${formatToolArgs(line.parameters)})`,
            id: line.tool_id,
            input: line.parameters,
            name: line.tool_name,
          },
        },
      ];
    }

    if (line.type === "tool_result" && typeof line.tool_id === "string") {
      return [
        {
          type: "openwiki",
          event: {
            type: "tool_end",
            id: line.tool_id,
            name: "tool",
            status: line.status === "error" ? "error" : "finished",
          },
        },
      ];
    }

    if (line.type === "error") {
      return [
        {
          type: "openwiki",
          event: {
            type: "debug",
            message: `ibm-bob ${
              typeof line.severity === "string" ? line.severity : "error"
            }: ${typeof line.message === "string" ? line.message : "unknown"}`,
          },
        },
      ];
    }

    if (line.type === "result") {
      const ok = line.status === "success";
      const error = isRecord(line.error) ? line.error : undefined;

      return [
        {
          type: "result",
          ok,
          errorMessage: ok
            ? undefined
            : typeof error?.message === "string" && error.message.length > 0
              ? error.message
              : `IBM Bob run ended with ${String(line.status ?? "an unknown error")}.`,
        },
      ];
    }

    return [];
  },
};

function parseInitEvent(line: Record<string, unknown>): AgentCliEvent[] {
  const events: AgentCliEvent[] = [];

  if (typeof line.session_id === "string" && line.session_id.length > 0) {
    events.push({ type: "session", sessionId: line.session_id });
  }

  events.push({
    type: "openwiki",
    event: {
      type: "debug",
      message: `ibm-bob session initialized model=${
        typeof line.model === "string" ? line.model : "unknown"
      }`,
    },
  });

  return events;
}

function parseMessageEvent(line: Record<string, unknown>): AgentCliEvent[] {
  if (
    line.role !== "assistant" ||
    typeof line.content !== "string" ||
    line.content.length === 0
  ) {
    return [];
  }

  return [
    {
      type: "openwiki",
      event: { source: "main", type: "text", text: line.content },
    },
  ];
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run test/ibm-bob-adapter.test.ts`
Expected: PASS (13 tests).

Run: `pnpm test && pnpm run lint && pnpm run build`
Expected: green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/engines/ibm-bob.ts test/ibm-bob-adapter.test.ts
git commit -m "feat: add the IBM Bob (Bob Shell) agent-cli adapter

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: Register the adapter and prove the runner pipeline end-to-end

**Files:**
- Modify: `src/agent/engines/index.ts`
- Test: `test/agent-cli-runner.test.ts`

**Interfaces:**
- Consumes: `ibmBobAdapter` from `./ibm-bob.js` (Task 3); `IBM_BOB_BINARY_ENV_KEY`, `getAgentCliProviderConfig("ibm-bob")` (Task 1); `runAgentCli`, `writeStub`, `baseSpec`, `savedEnv` scaffolding already in the test file.
- Produces: `getAgentCliAdapter("ibm-bob")` returns `ibmBobAdapter`. This is the final wiring — after this task, `OPENWIKI_PROVIDER=ibm-bob` works through the whole app.

- [ ] **Step 1: Write the failing tests**

In `test/agent-cli-runner.test.ts`:

(a) Extend the imports: add `IBM_BOB_BINARY_ENV_KEY` to the `../src/constants.ts` import, and add:

```ts
import { ibmBobAdapter } from "../src/agent/engines/ibm-bob.ts";
```

(b) Add `IBM_BOB_BINARY_ENV_KEY` to the env-key save/clear list in `beforeEach` (the `for (const key of [...])` array), so the suite isolates it like the claude key.

(c) Add a Bob stub next to the other stub constants:

```ts
const BOB_SUCCESS_STUB = `#!/usr/bin/env node
if (process.argv.includes("--version")) {
  console.log("1.0.3-stub");
  process.exit(0);
}
let input = "";
process.stdin.on("data", (chunk) => (input += chunk));
process.stdin.on("end", () => {
  console.log(JSON.stringify({ type: "init", timestamp: "t", session_id: "bob-session", model: "bob-model" }));
  console.log(JSON.stringify({ type: "message", timestamp: "t", role: "assistant", content: "stdin-bytes:" + input.length, delta: true }));
  console.log(JSON.stringify({ type: "tool_use", timestamp: "t", tool_name: "write_to_file", tool_id: "bob-tool-1", parameters: { path: "openwiki/quickstart.md" } }));
  console.log(JSON.stringify({ type: "tool_result", timestamp: "t", tool_id: "bob-tool-1", status: "success", output: "ok" }));
  console.log(JSON.stringify({ type: "result", timestamp: "t", status: "success", stats: {} }));
});
`;
```

(d) Update the registry test to also assert the Bob adapter:

```ts
describe("getAgentCliAdapter", () => {
  test("returns the registered adapters and rejects api providers", () => {
    expect(getAgentCliAdapter("claude-code")).toBe(claudeCodeAdapter);
    expect(getAgentCliAdapter("ibm-bob")).toBe(ibmBobAdapter);
    expect(() => getAgentCliAdapter("openai")).toThrow(/openai/);
  });
});
```

(e) Add a Bob pipeline test inside `describe("runAgentCli", ...)`:

```ts
  test("runs the ibm-bob adapter end-to-end with composed stdin and patched tool names", async () => {
    process.env[IBM_BOB_BINARY_ENV_KEY] = await writeStub(
      stubDir,
      "bob-stub",
      BOB_SUCCESS_STUB,
    );
    const events: OpenWikiRunEvent[] = [];

    const outcome = await runAgentCli(
      ibmBobAdapter,
      getAgentCliProviderConfig("ibm-bob"),
      baseSpec,
      { onEvent: (event) => events.push(event) },
    );

    expect(outcome.sessionId).toBe("bob-session");
    const expectedBytes = `${baseSpec.systemPrompt}\n\n${baseSpec.prompt}`
      .length;
    const text = events.find((event) => event.type === "text");
    expect(text).toMatchObject({ text: `stdin-bytes:${expectedBytes}` });
    const toolEnd = events.find((event) => event.type === "tool_end");
    expect(toolEnd).toMatchObject({
      id: "bob-tool-1",
      name: "write_to_file",
      status: "finished",
    });
  });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/agent-cli-runner.test.ts`
Expected: FAIL — `getAgentCliAdapter("ibm-bob")` throws `No agent CLI adapter is registered for ibm-bob.`

- [ ] **Step 3: Register the adapter**

Replace `src/agent/engines/index.ts` with:

```ts
import type { OpenWikiProvider } from "../../constants.js";
import { claudeCodeAdapter } from "./claude-code.js";
import { ibmBobAdapter } from "./ibm-bob.js";
import type { AgentCliAdapter } from "./types.js";

const adapters: Partial<Record<OpenWikiProvider, AgentCliAdapter>> = {
  "claude-code": claudeCodeAdapter,
  "ibm-bob": ibmBobAdapter,
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

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run test/agent-cli-runner.test.ts`
Expected: PASS (all existing + 2 new/updated).

Run: `pnpm test && pnpm run lint && pnpm run build`
Expected: green.

- [ ] **Step 5: Commit**

```bash
git add src/agent/engines/index.ts test/agent-cli-runner.test.ts
git commit -m "feat: register the ibm-bob adapter in the engine registry

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: README and wiki documentation

**Files:**
- Modify: `README.md` (the "Use your coding-agent subscription (no API key)" section, lines 75–98)
- Modify: `openwiki/cli/usage.md`, `openwiki/quickstart.md`, `openwiki/agent/workflow.md`, `openwiki/architecture/overview.md`, `openwiki/operations/credentials-and-updates.md` — every sentence that presents `claude-code` as the only agent-cli adapter.

**Interfaces:**
- Consumes: the exact names shipped in Tasks 1–4: provider id `ibm-bob`, label `IBM Bob (subscription)`, env key `OPENWIKI_IBM_BOB_BINARY`, binary `bob`.
- Produces: user-facing docs; no code.

- [ ] **Step 1: Update the README section**

Replace lines 77–79 ("If you already pay ... The first supported agent is Claude Code:") with:

```markdown
If you already pay for a subscription coding agent, OpenWiki can delegate
documentation runs to it instead of calling a metered API. Two agents are
supported: Claude Code and IBM Bob (Bob Shell).

**Claude Code:**
```

After the existing Claude notes (line 96, "- LangSmith tracing does not apply to delegated runs."), insert the Bob block before the POSIX bullet:

```markdown

**IBM Bob (Bob Shell):**

```bash
OPENWIKI_PROVIDER=ibm-bob
OPENWIKI_MODEL_ID=default   # or any model id your Bob backend accepts
```

- Install Bob Shell (`curl -fsSL https://bob.ibm.com/download/bobshell.sh | bash`),
  then run `bob` once in the target repository to complete the IBMid login
  and trust the folder when prompted — Bob refuses write-enabled headless
  runs in untrusted folders.
- Runs execute through `bob` in headless mode (`--approval-mode auto_edit`
  with a read-only-git shell allowlist), using your existing login. No API
  key is stored.
- Set `OPENWIKI_IBM_BOB_BINARY` to point at a non-default binary location.
```

Then generalize the POSIX bullet (line 97–98) from "for the claude-code provider" to:

```markdown
- macOS and Linux only for now: process-group management and binary
  resolution for the subscription providers are POSIX-specific.
```

- [ ] **Step 2: Update the wiki pages**

Search the five wiki files for `claude-code` and `Claude Code` (e.g. `grep -rn "claude-code\|Claude Code" openwiki/`) and update each spot that enumerates agent-cli adapters so it lists both `claude-code` and `ibm-bob`. Known spots and their edits:

- `openwiki/cli/usage.md`: the provider list/table gains an `ibm-bob` row — label `IBM Bob (subscription)`, kind `agent-cli`, binary `bob`, override key `OPENWIKI_IBM_BOB_BINARY`; the "how to add a provider" note stays generic.
- `openwiki/quickstart.md`: provider overview sentences mentioning "claude-code" as the delegated option now say "claude-code and ibm-bob".
- `openwiki/agent/workflow.md`: "currently `claude-code`" becomes "currently `claude-code` and `ibm-bob`"; where Claude flags are described, add one sentence: "The ibm-bob adapter uses Bob Shell's `--approval-mode auto_edit`, pins `--chat-mode advanced`, and prepends the system prompt to the stdin payload because Bob has no append-system-prompt flag."
- `openwiki/architecture/overview.md`: "the Claude Code adapter" phrasing becomes "the Claude Code and IBM Bob adapters"; the extension note stays as-is.
- `openwiki/operations/credentials-and-updates.md`: the binary-override and diagnostics lists gain `OPENWIKI_IBM_BOB_BINARY` next to `OPENWIKI_CLAUDE_CODE_BINARY`.

Do not touch `docs/superpowers/` (historical planning docs) or model-id mentions of Claude under the `anthropic`/`openrouter` API providers.

- [ ] **Step 3: Verify**

Run: `pnpm test && pnpm run lint && pnpm run build`
Expected: green (docs-only change; lint covers markdown via prettier if configured — if `pnpm run format:check` exists, run it too).

Run: `grep -rn "first supported agent" README.md openwiki/`
Expected: no matches (the singular phrasing is gone).

- [ ] **Step 4: Commit**

```bash
git add README.md openwiki/
git commit -m "docs: document the ibm-bob subscription provider

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: Live verification against the installed Bob Shell (controller-led — NOT a subagent task)

This task is executed by the session controller with the user, because it
needs the user's IBMid login and consumes their Bob subscription quota.
Prerequisite: the user re-authenticates Bob (`bob` interactively or `! bob`)
— the current SSO session is expired — and trusts the fixture repo folder.

**Files:** none in-repo (fixture repo in the session scratchpad; findings feed fixes back into Tasks 3–5 files if the live schema deviates).

- [ ] **Step 1: Probe the real stream (schema check)**

In a trusted scratch directory:

```bash
bob --output-format stream-json "Reply with exactly the word: PONG" > bob-probe.ndjson 2>bob-probe.stderr; echo "exit=$?"
```

Compare every line against the fixtures in `test/ibm-bob-adapter.test.ts` field-for-field (`init.session_id` is a uuid; `message` deltas; `result.status`). If any shape differs, fix `parseEvent` + fixtures via TDD before proceeding.

- [ ] **Step 2: Confirm write behavior and tool names**

In a trusted fixture repo, run a prompt that forces a file write under `--approval-mode auto_edit` (no `--yolo`) and confirm (a) the file is written headless, (b) the `tool_use.tool_name` values for read/write/shell tools, adjusting `IBM_BOB_ALLOWED_TOOLS` if the runtime names differ from `run_shell_command`. If `auto_edit` cannot write headless, decision point with the user: switch `buildArgs` to `--yolo` (Bob still confines writes to the start directory) — do not change silently.

- [ ] **Step 3: Confirm stdin-only invocation and resume**

```bash
printf 'Reply with exactly: PING' | bob --output-format stream-json > /tmp/bob-stdin.ndjson
```

Expected: one-shot run completes (no interactive hang). Then re-run with `--resume <session_id from the init event>` and confirm the session continues.

- [ ] **Step 4: End-to-end OpenWiki run**

```bash
pnpm run build
cd <fixture-repo>   # trusted in Bob
OPENWIKI_PROVIDER=ibm-bob node <repo>/dist/cli.js --print --init
```

Expected: `openwiki/` docs written, `.last-update.json` recorded, no API key touched. Follow with an interactive session and a follow-up message to confirm `--resume` continuity, and a dogfood `--update` on this repository.

- [ ] **Step 5: Negative paths**

- Untrusted folder: run in a non-trusted dir; expect the run to fail with Bob's "Cannot enable privileged approval modes in an untrusted folder" (or equivalent) visible in the stderr tail of OpenWiki's error.
- Logged out: after the user logs out (or with auth expired), expect the failure message to include the installHint (the stderr contains "authentication", which the runner's login-hint regex matches).

- [ ] **Step 6: Record outcomes**

Fold any adapter changes back through Tasks 3–4 test files (TDD), commit, and note verified behaviors in the PR description.
