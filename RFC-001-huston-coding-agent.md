# RFC-001: Huston Coding Agent

**Status:** Draft  
**Author:** —  
**Date:** 2026-03-01  
**Framework:** [Glove](https://glove.dterminal.net) (`glove-core` only)

---

## 1. Overview

Huston's first agent is a coding agent with feature parity to [OpenCode](https://github.com/opencode-ai/opencode). It runs as a **server-side `glove-core` process** — no `glove-react`, no `glove-next`. Frontends (CLI, TUI, web, desktop) connect over a transport layer and subscribe to events. The agent owns all tool execution, persistence, and model interaction. Frontends are thin renderers.

### Why glove-core only?

OpenCode bundles its agent, tools, and UI into one binary. Huston separates them. The agent is a headless server. This gives us:

- **One agent, many frontends** — a TUI, a web UI, a VS Code extension, all talking to the same session.
- **No Vercel AI SDK** — `glove-core` uses pluggable `ModelAdapter`s. Local models (Ollama, llama.cpp, LM Studio) work via `OpenAICompatAdapter` out of the box.
- **Server-side tools** — file I/O, bash, git all run in the agent process. No client-side tool execution, no browser sandboxing.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Huston Server                        │
│                                                         │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Glove    │  │  SqliteStore  │  │  Displaymanager  │ │
│  │  (Agent)   │──│  (sessions)   │  │  (slot stack)    │ │
│  └─────┬─────┘  └──────────────┘  └────────┬─────────┘ │
│        │                                    │           │
│  ┌─────┴──────────────────────────┐         │           │
│  │         Tool Registry          │         │           │
│  │  bash · read · write · edit    │         │           │
│  │  multiedit · glob · grep · ls  │         │           │
│  │  webfetch · task · todo        │         │           │
│  │  question · skill · patch      │         │           │
│  └────────────────────────────────┘         │           │
│                                             │           │
│  ┌──────────────────────────────────────────┴─────────┐ │
│  │              Transport Layer                        │ │
│  │  SubscriberAdapter → WebSocket / stdio / IPC       │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    ┌────┴───┐   ┌──────┴──────┐  ┌───┴────┐
    │  CLI   │   │   Web UI    │  │ VS Code│
    │ (Ink)  │   │  (React)    │  │  ext   │
    └────────┘   └─────────────┘  └────────┘
```

### 2.1 Session Management

Each coding session maps to a `SqliteStore` instance keyed by session ID. The store tracks:

- Message history (full, uncompacted — frontends render the whole timeline)
- Token and turn counters (for compaction triggers)
- Tasks (todo list)
- Permissions (per-tool grant/deny/unset)
- Working directory (the project root)
- Session name (auto-generated or user-set)

```typescript
import { SqliteStore } from "glove-core";

const store = new SqliteStore({
  dbPath: "~/.huston/sessions.db",  // single DB, multiple sessions
  sessionId: "session-abc123",
});
store.setWorkingDir("/home/user/my-project");
```

Sessions are resumable. `SqliteStore.listSessions(dbPath)` provides session history for the frontend.

### 2.2 Agent Construction

```typescript
import { Glove, SqliteStore, Displaymanager, createAdapter } from "glove-core";

function createSession(sessionId: string, config: HustonConfig): IGloveRunnable {
  const store = new SqliteStore({ dbPath: config.dbPath, sessionId });
  const model = createAdapter({
    provider: config.provider,       // "anthropic" | "openai" | "openrouter" | etc
    model: config.model,
    apiKey: config.apiKey,
    stream: true,
  });
  const display = new Displaymanager();
  const cwd = store.getWorkingDir() ?? config.defaultCwd;

  const glove = new Glove({
    store,
    model,
    displayManager: display,
    systemPrompt: buildSystemPrompt(cwd, config),
    compaction_config: {
      compaction_instructions: COMPACTION_PROMPT,
      max_turns: 120,
      compaction_context_limit: 100_000,
    },
  });

  // Register all tools
  for (const tool of createToolRegistry(cwd, store, display, config)) {
    glove.fold(tool);
  }

  // Bridge events to transport
  glove.addSubscriber(createTransportSubscriber(sessionId, config.transport));

  return glove.build();
}
```

### 2.3 Transport Layer

The transport is a `SubscriberAdapter` that bridges Glove events to whatever protocol the frontend uses. The server exposes session management and the display stack over the same transport.

**Protocol-agnostic event contract:**

```typescript
// Server → Frontend
type ServerEvent =
  | { type: "text_delta"; text: string }
  | { type: "tool_use"; id: string; name: string; input: unknown }
  | { type: "tool_use_result"; tool_name: string; call_id: string; result: ToolResultData }
  | { type: "model_response_complete"; text: string; tool_calls: ToolCall[]; tokens_in: number; tokens_out: number }
  | { type: "compaction_start"; current_token_consumption: number }
  | { type: "compaction_end"; current_token_consumption: number; summary_message: Message }
  | { type: "slot_push"; slot: SlotData; _bubble?: BubbleContext }   // display stack push (tagged if from child)
  | { type: "slot_remove"; slotId: string }                          // display stack remove
  | { type: "slot_clear" }                                           // display stack clear
  | { type: "permission_request"; toolName: string; input: unknown; callId: string; _bubble?: BubbleContext }
  | { type: "plan_mode_changed"; active: boolean }                   // plan mode state change
  | { type: "session_title"; title: string }                         // auto-generated or user-set title
  | { type: "session_summary"; summary: string }                     // session summary
  | { type: "session_list_result"; sessions: SessionInfo[] }         // response to session_list
  | { type: "error"; message: string }
  // Child agent streaming events (optional — frontend can show sub-agent activity)
  | { type: "child_text_delta"; text: string; _childId: string; _agentType: string }
  | { type: "child_tool_use"; id: string; name: string; input: unknown; _childId: string; _agentType: string }

// Frontend → Server
type ClientEvent =
  | { type: "message"; text: string; images?: ContentPart[] }
  | { type: "abort" }
  | { type: "slot_resolve"; slotId: string; value: unknown }
  | { type: "slot_reject"; slotId: string; reason?: string }
  | { type: "permission_response"; callId: string; granted: boolean; remember: boolean }
  | { type: "set_model"; provider: string; model: string }
  | { type: "session_list" }
  | { type: "session_resume"; sessionId: string }
  | { type: "session_rename"; title: string }
  | { type: "session_delete"; sessionId: string }
  | { type: "session_summarize" }

// Bubble context — present on slots/events that originated from a child agent
interface BubbleContext {
  childId: string;      // child session ID (e.g. "task-abc123")
  agentType: string;    // agent type (e.g. "explore", "plan")
}

interface SessionInfo {
  id: string;
  name: string | null;
  workingDir: string | null;
  createdAt: string;
  updatedAt: string;
}
```

**Transport implementations (Phase 1):**

| Transport | Use case | Implementation |
|-----------|----------|----------------|
| WebSocket | Web UI, desktop app | `ws` or native `WebSocket` server |
| stdio | CLI embedding, pipe to other tools | JSON-line protocol over stdin/stdout |

The `SubscriberAdapter` + `Displaymanager.subscribe()` give us everything we need. The subscriber bridges `text_delta`, `tool_use`, `tool_use_result`, and `model_response_complete` events. The display manager subscription bridges slot pushes, resolves, and clears.

```typescript
function createTransportSubscriber(sessionId: string, transport: Transport): SubscriberAdapter {
  return {
    async record(event_type, data) {
      transport.send(sessionId, { type: event_type, ...data });
    },
  };
}

// Display stack bridge
display.subscribe((stack) => {
  transport.send(sessionId, { type: "slot_update", stack });
});
```

### 2.4 Permission System

OpenCode gates destructive tools (bash, write, edit, patch) behind user approval. Glove supports this natively via `requiresPermission` on tool registration and `getPermission`/`setPermission` on the store.

When a tool with `requiresPermission: true` is called:
1. Glove checks `store.getPermission(toolName)`
2. If `"unset"`, it pushes a permission request slot via `Displaymanager.pushAndWait`
3. The transport bridges this as a `permission_request` event to the frontend
4. Frontend shows the approval UI and sends back `permission_response`
5. The slot resolves, tool executes (or is denied)

If the user selects "always allow", `store.setPermission(toolName, "granted")` is called and future calls skip the prompt.

---

## 3. Tool Registry

### 3.1 Feature Parity Matrix

Every OpenCode tool mapped to its Huston equivalent:

| OpenCode Tool | Huston Tool | Glove Mapping | Notes |
|---------------|-------------|---------------|-------|
| `bash` | `bash` | `fold({ requiresPermission: true })` | Persistent shell session via `node-pty` or `child_process` |
| `read` | `read` | `fold({})` | Line-numbered output, offset/limit pagination, image/PDF as attachments |
| `write` | `write` | `fold({ requiresPermission: true })` | Full file write, requires prior `read` |
| `edit` | `edit` | `fold({ requiresPermission: true })` | Exact string replacement, unique match required |
| `multiedit` | `multiedit` | `fold({ requiresPermission: true })` | Atomic multi-edit on single file |
| `apply_patch` | `apply_patch` | `fold({ requiresPermission: true })` | GPT-4.1+ custom diff format |
| `glob` | `glob` | `fold({})` | Fast pattern matching via `fast-glob` |
| `grep` | `grep` | `fold({})` | Regex search via `ripgrep` (child process) |
| `list` | `list` | `fold({})` | Directory listing |
| `webfetch` | `webfetch` | `fold({})` | URL fetch with markdown/text/html conversion |
| `websearch` | `websearch` | `fold({})` | Exa AI search (conditional) |
| `codesearch` | `codesearch` | `fold({})` | Exa Code API search (conditional) |
| `task` | `task` | `fold({})` | Spawn sub-agent (child Glove session) |
| `batch` | `batch` | `fold({})` | Parallel tool execution (1-25 calls) |
| `question` | `question` | `fold({})` with `pushAndWait` | Ask user a question, block until answered |
| `todo_read` | Built-in | `store.getTasks()` | Glove's native task tool auto-registers |
| `todo_write` | Built-in | `store.addTasks()` / `store.updateTask()` | Glove's native task tool |
| `skill` | `skill` | `fold({})` | Load markdown knowledge from `.huston/skills/` |
| `lsp` | `lsp` | `fold({})` | LSP integration (Phase 2 — experimental) |
| `plan_enter` | `plan_enter` | `fold({})` with `pushAndWait` | Switch to read-only plan mode |
| `plan_exit` | `plan_exit` | `fold({})` with `pushAndWait` | Exit plan mode, switch to build |

### 3.2 Tool Implementation Pattern

All tools follow the same structure. They receive the working directory at construction time and resolve paths against it.

```typescript
// tools/bash.ts
import { z } from "zod";
import type { GloveFoldArgs } from "glove-core";

export function createBashTool(cwd: string): GloveFoldArgs<BashInput> {
  return {
    name: "bash",
    description: BASH_DESCRIPTION,  // loaded from bash.txt equivalent
    inputSchema: z.object({
      command: z.string().describe("The shell command to run"),
      workdir: z.string().optional().describe("Working directory override"),
      timeout: z.number().optional().describe("Timeout in ms (default 120000)"),
      description: z.string().optional().describe("5-10 word summary of what this does"),
    }),
    requiresPermission: true,
    async do(input) {
      const execDir = input.workdir
        ? resolve(cwd, input.workdir)
        : cwd;
      const timeout = input.timeout ?? 120_000;

      const { stdout, stderr, exitCode } = await execCommand(input.command, {
        cwd: execDir,
        timeout,
      });

      // Truncate output like OpenCode does
      const truncated = truncateOutput(stdout, { maxLines: 2000, maxBytes: 100_000 });

      return {
        status: exitCode === 0 ? "success" : "error",
        data: formatBashOutput(truncated, stderr, exitCode),
        message: exitCode !== 0 ? `Command exited with code ${exitCode}` : undefined,
        renderData: { command: input.command, exitCode, truncated: truncated.wasTruncated },
      };
    },
  };
}
```

### 3.3 The `question` Tool — pushAndWait Bridge

This is the key tool that bridges agent ↔ user interaction. OpenCode's `question` tool presents options to the user and blocks until they respond. In Glove, this maps directly to `Displaymanager.pushAndWait`.

```typescript
export function createQuestionTool(): GloveFoldArgs<QuestionInput> {
  return {
    name: "question",
    description: "Ask the user a question. Use to gather preferences, clarify ambiguity, or get decisions.",
    inputSchema: z.object({
      question: z.string().describe("The question to ask"),
      options: z.array(z.string()).describe("Array of option labels"),
      multiple: z.boolean().optional().describe("Allow multiple selections (default false)"),
      custom: z.boolean().optional().describe("Allow free-text answer (default true)"),
    }),
    async do(input, display) {
      const result = await display.pushAndWait({
        renderer: "question",
        input: {
          question: input.question,
          options: input.options,
          multiple: input.multiple ?? false,
          custom: input.custom ?? true,
        },
      });

      return {
        status: "success",
        data: `User answered: ${JSON.stringify(result)}`,
        renderData: { question: input.question, answer: result },
      };
    },
  };
}
```

The frontend receives this as a `slot_push` event with `renderer: "question"`, renders the appropriate UI (buttons in a TUI, a dropdown in web), and sends back `slot_resolve` with the user's choice.

### 3.4 The `task` Tool — Sub-Agent Spawning & Bubble-Up

OpenCode's `task` tool spawns a child agent session. In Huston, this creates a nested `Glove` instance with a scoped `SqliteStore` session and a limited tool set.

**The bubble-up problem:** A sub-agent is headless — it has no frontend connection. But sub-agents can register tools that need user interaction (`question`, permission prompts, etc.). If a child's `Displaymanager` pushes a `pushAndWait` slot, it deadlocks — nobody is listening. The child's display stack needs to bubble up through the parent's `Displaymanager` so the real frontend can resolve it.

**Solution: Display proxy.** The parent passes its own `Displaymanager` (or a proxy wrapping it) to the child. When the child pushes a slot, it surfaces on the parent's stack, the frontend sees it, and the resolve flows back down.

```
┌─────────────────────────────────────────────────┐
│                 Parent Agent                     │
│                                                  │
│  Displaymanager (parent)                         │
│       ▲                                          │
│       │ subscribe()                              │
│       ▼                                          │
│  Transport → Frontend                            │
│       ▲                                          │
│       │ slot events bubble up                    │
│       │                                          │
│  ┌────┴──────────────────────────────────┐       │
│  │           Child Agent                  │       │
│  │                                        │       │
│  │  DisplayProxy ──────► parent.display   │       │
│  │  (wraps parent Displaymanager)         │       │
│  │                                        │       │
│  │  question tool → pushAndWait           │       │
│  │       └──► proxy.pushAndWait           │       │
│  │              └──► parent.pushAndWait   │       │
│  │                     └──► frontend      │       │
│  │                          resolve ──►   │       │
│  │                     ◄── value          │       │
│  │              ◄── value                 │       │
│  │       ◄── value                        │       │
│  │  question tool returns                 │       │
│  └────────────────────────────────────────┘       │
└──────────────────────────────────────────────────┘
```

**DisplayProxy** wraps the parent's `Displaymanager` and tags every slot with the child's identity so the frontend (and parent subscriber) can distinguish child slots from parent slots:

```typescript
import { Displaymanager } from "glove-core";

interface DisplayProxyOpts {
  parentDisplay: Displaymanager;
  childId: string;          // e.g. "task-abc123"
  agentType: string;        // e.g. "explore"
}

/**
 * Proxies a child agent's display operations through the parent's Displaymanager.
 * Implements the same interface as Displaymanager so Glove accepts it.
 * Every slot pushed gets tagged with { _childId, _agentType } so the frontend
 * can render child interactions with appropriate context (e.g. "Sub-agent is asking:").
 */
function createDisplayProxy(opts: DisplayProxyOpts): Displaymanager {
  const { parentDisplay, childId, agentType } = opts;

  // Create a real Displaymanager that delegates to the parent
  const proxy = new Displaymanager();

  // Override pushAndWait to go through the parent
  const originalPushAndWait = proxy.pushAndWait.bind(proxy);
  proxy.pushAndWait = async (slot) => {
    // Tag the slot so the frontend knows this came from a child
    const taggedSlot = {
      ...slot,
      input: {
        ...(typeof slot.input === "object" ? slot.input : { value: slot.input }),
        _bubble: { childId, agentType },
      },
    };
    // Push onto PARENT's stack — this is what the frontend subscribes to
    return parentDisplay.pushAndWait(taggedSlot);
  };

  // Override pushAndForget similarly
  const originalPushAndForget = proxy.pushAndForget.bind(proxy);
  proxy.pushAndForget = async (slot) => {
    const taggedSlot = {
      ...slot,
      input: {
        ...(typeof slot.input === "object" ? slot.input : { value: slot.input }),
        _bubble: { childId, agentType },
      },
    };
    return parentDisplay.pushAndForget(taggedSlot);
  };

  return proxy;
}
```

The frontend checks for `_bubble` in slot data to render child-originated interactions differently — e.g. prefixing with "Sub-agent (explore) is asking:" or nesting the UI.

**The task tool itself:**

```typescript
export function createTaskTool(
  parentCwd: string,
  parentDisplay: Displaymanager,
  config: HustonConfig,
  agentRegistry: AgentRegistry,
): GloveFoldArgs<TaskInput> {
  return {
    name: "task",
    description: "Launch a sub-agent for complex, multistep tasks. Supports parallel execution.",
    inputSchema: z.object({
      subagent_type: z.string().describe("Which agent type to launch (e.g. 'explore', 'code')"),
      description: z.string().describe("Brief task description"),
      prompt: z.string().describe("Full prompt for the sub-agent"),
      task_id: z.string().optional().describe("Resume an existing sub-agent session"),
    }),
    async do(input) {
      const agentConfig = agentRegistry.get(input.subagent_type);
      if (!agentConfig) {
        return { status: "error", data: null, message: `Unknown agent type: ${input.subagent_type}` };
      }

      const childSessionId = input.task_id ?? `task-${randomId()}`;
      const childStore = new SqliteStore({ dbPath: config.dbPath, sessionId: childSessionId });

      // Model selection: use agent-specific override, or fall back to parent's model
      const modelConfig = agentConfig.modelOverride ?? { provider: config.provider, model: config.model };
      const childModel = createAdapter({ ...modelConfig, stream: false });

      // CRITICAL: proxy display through parent so slots bubble up to the frontend
      const childDisplay = createDisplayProxy({
        parentDisplay,
        childId: childSessionId,
        agentType: input.subagent_type,
      });

      const child = new Glove({
        store: childStore,
        model: childModel,
        displayManager: childDisplay,  // proxied — slots bubble to parent
        systemPrompt: agentConfig.systemPrompt,
        compaction_config: { compaction_instructions: COMPACTION_PROMPT, max_turns: 50 },
      });

      // Sub-agents get a restricted tool set (read-only for explore, full for code)
      for (const tool of agentConfig.tools(parentCwd)) {
        child.fold(tool);
      }

      // Bridge child events to parent transport (prefixed for frontend filtering)
      child.addSubscriber({
        async record(event_type, data) {
          parentTransport.send(childSessionId, {
            type: event_type,
            _childId: childSessionId,
            _agentType: input.subagent_type,
            ...data,
          });
        },
      });

      const agent = child.build();

      // Run to completion — sub-agent output becomes the tool result
      // If the child calls `question`, pushAndWait bubbles to the parent's
      // display stack → frontend resolves → value flows back → child continues
      await agent.processRequest(input.prompt);
      const messages = await childStore.getMessages();
      const lastAgentMsg = messages.filter(m => m.sender === "agent").pop();

      return {
        status: "success",
        data: lastAgentMsg?.text ?? "Sub-agent completed with no output.",
        renderData: { subagent_type: input.subagent_type, task_id: childSessionId },
      };
    },
  };
}
```

**What bubbles up:**

| Child action | Bubble mechanism | Frontend sees |
|---|---|---|
| `question` tool → `pushAndWait` | Proxy delegates to `parentDisplay.pushAndWait` | `slot_push` with `_bubble: { childId, agentType }` |
| Permission request (`requiresPermission` tool) | Glove internally calls `pushAndWait` on the display, which is proxied | `permission_request` with child context |
| `pushAndForget` (status cards, progress) | Proxy delegates to `parentDisplay.pushAndForget` | `slot_push` with `_bubble` tag — frontend can show in a sub-section |
| `text_delta`, `tool_use`, etc. | Child subscriber → transport with `_childId` prefix | Streaming events tagged with child ID — frontend can show sub-agent activity |

**Nested sub-agents:** If a child spawns its own sub-agent (e.g. plan agent launches explore agents), the same proxy chain works. The grandchild gets a proxy pointing at the child's display, which is itself a proxy pointing at the parent's display. Slots bubble all the way up. The `_bubble` tag becomes the deepest child's info, which is sufficient — the frontend doesn't need to know the full ancestry.

**Parallel sub-agents:** When the parent launches multiple `task` calls in parallel (or via `batch`), each child gets its own proxy. Multiple `pushAndWait` slots can be active simultaneously on the parent's display stack. The frontend renders them all; each resolves independently back to its respective child.

### 3.5 The `batch` Tool — Parallel Execution

Wraps multiple tool calls for concurrent execution:

```typescript
export function createBatchTool(toolRegistry: Map<string, GloveFoldArgs<any>>): GloveFoldArgs<BatchInput> {
  return {
    name: "batch",
    description: "Execute 1-25 independent tool calls concurrently. All start in parallel.",
    inputSchema: z.object({
      calls: z.array(z.object({
        tool: z.string().describe("Tool name"),
        parameters: z.record(z.unknown()).describe("Tool parameters"),
      })).min(1).max(25),
    }),
    async do(input) {
      const results = await Promise.allSettled(
        input.calls.map(async (call) => {
          const tool = toolRegistry.get(call.tool);
          if (!tool) return { status: "error" as const, data: null, message: `Unknown tool: ${call.tool}` };
          const parsed = tool.inputSchema.safeParse(call.parameters);
          if (!parsed.success) return { status: "error" as const, data: null, message: parsed.error.message };
          return tool.do(parsed.data, dummyDisplay);
        })
      );

      const formatted = results.map((r, i) => ({
        tool: input.calls[i].tool,
        result: r.status === "fulfilled" ? r.value : { status: "error", data: null, message: String(r.reason) },
      }));

      return {
        status: "success",
        data: JSON.stringify(formatted, null, 2),
      };
    },
  };
}
```

### 3.6 The `skill` Tool — Contextual Knowledge Injection

```typescript
export function createSkillTool(skillDirs: string[]): GloveFoldArgs<SkillInput> {
  return {
    name: "skill",
    description: "Load a named skill to inject contextual knowledge into the conversation.",
    inputSchema: z.object({
      name: z.string().describe("Skill identifier to load"),
    }),
    async do(input) {
      for (const dir of skillDirs) {
        const skillPath = resolve(dir, `${input.name}.md`);
        if (existsSync(skillPath)) {
          const content = await readFile(skillPath, "utf-8");
          return { status: "success", data: content };
        }
      }
      return { status: "error", data: null, message: `Skill not found: ${input.name}` };
    },
  };
}
```

Skill directories searched in order: `.huston/skills/` (project-local) → `~/.huston/skills/` (global).

### 3.7 Git Safety

All file-mutating tools (`write`, `edit`, `multiedit`, `apply_patch`, `bash`) trigger a git snapshot before execution. This is implemented as a pre-execution hook in the tool registry factory:

```typescript
function withGitSnapshot<I>(tool: GloveFoldArgs<I>, cwd: string): GloveFoldArgs<I> {
  const originalDo = tool.do;
  return {
    ...tool,
    async do(input, display) {
      await gitSnapshot(cwd);  // git stash-like checkpoint
      return originalDo(input, display);
    },
  };
}
```

Git policy (enforced in bash tool description + system prompt):
- Never force push
- Never amend commits that have been pushed
- Never skip hooks
- Only commit if user explicitly asks

---

## 4. Sub-Agent Registry

OpenCode defines several hidden agent types used by the `task` tool. Huston replicates this as a typed registry:

```typescript
interface AgentConfig {
  id: string;
  systemPrompt: string;
  tools: (cwd: string) => GloveFoldArgs<any>[];
  modelOverride?: { provider: string; model: string };  // cheaper/faster model for this agent type
}

const AGENTS: AgentConfig[] = [
  {
    id: "explore",
    systemPrompt: EXPLORE_PROMPT,  // read-only codebase navigation
    tools: (cwd) => [
      createGlobTool(cwd),
      createGrepTool(cwd),
      createReadTool(cwd),
      createListTool(cwd),
    ],
    // Example: use a cheaper model for exploration since it's read-only
    // modelOverride: { provider: "anthropic", model: "claude-haiku-4-5-20251001" },
  },
  {
    id: "code",
    systemPrompt: CODE_PROMPT,  // full coding agent
    tools: (cwd) => createFullToolSet(cwd),
    // No override — uses parent's model by default
  },
  {
    id: "plan",
    systemPrompt: PLAN_PROMPT,  // planning only, read + explore tools
    tools: (cwd) => [
      createReadTool(cwd),
      createGlobTool(cwd),
      createGrepTool(cwd),
      createListTool(cwd),
      createTaskTool(cwd, parentDisplay, config, agentRegistry),  // can spawn explore sub-agents
    ],
  },
];
```

Users can define custom agents in `.huston/agents/` as markdown files with YAML frontmatter (matching OpenCode's `AGENTS.md` convention).

---

## 5. System Prompt Strategy

### 5.1 Per-Provider Prompts

OpenCode ships different system prompts per model family (Anthropic, GPT-4.x, Gemini, Qwen, etc). Huston does the same. The prompt is selected based on `config.provider` + `config.model`:

```typescript
function selectPromptTemplate(provider: string, model: string): string {
  if (model.includes("claude")) return ANTHROPIC_PROMPT;
  if (model.includes("gpt-5")) return CODEX_PROMPT;
  if (model.includes("gpt-") || model.includes("o1") || model.includes("o3")) return BEAST_PROMPT;
  if (model.includes("gemini")) return GEMINI_PROMPT;
  return DEFAULT_PROMPT;  // Qwen-derived, works for local models
}
```

### 5.2 Dynamic Environment Block

Appended to every system prompt:

```typescript
function environmentBlock(cwd: string, model: string, provider: string): string {
  return `
You are powered by ${model} via ${provider}.
Working directory: ${cwd}
Is git repo: ${isGitRepo(cwd) ? "yes" : "no"}
Platform: ${process.platform}
Today's date: ${new Date().toDateString()}
`.trim();
}
```

### 5.3 Runtime Injected Prompts

These are appended as system-reminder content mid-session:

| Trigger | Prompt | Purpose |
|---------|--------|---------|
| Plan mode entered | `PLAN_MODE_PROMPT` | Lock agent to read-only, enforce planning workflow |
| Plan mode exited | `BUILD_SWITCH_PROMPT` | Unlock file mutations |
| Max steps reached | `MAX_STEPS_PROMPT` | Force text-only response, summarize state |

### 5.4 Compaction Prompt

Used by Glove's built-in context compaction:

```typescript
const COMPACTION_PROMPT = `Summarize this conversation focusing on:
- What was accomplished
- What is currently being worked on and which files are involved
- What needs to be done next
- Key user constraints and preferences
- Important technical decisions made
Be comprehensive enough to continue working from this summary without re-reading history.`;
```

---

## 6. Configuration

### 6.1 Config File

`~/.huston/config.toml` (global) and `.huston/config.toml` (project-local, overrides global):

```toml
[model]
provider = "anthropic"
model = "claude-sonnet-4-20250514"
# api_key read from env: ANTHROPIC_API_KEY, OPENAI_API_KEY, etc.

[model.local]
# For local models — uses OpenAICompatAdapter
provider = "openai"
base_url = "http://localhost:11434/v1"  # Ollama
model = "qwen2.5-coder:32b"

[session]
db_path = "~/.huston/sessions.db"
max_turns = 120
compaction_context_limit = 100000

[transport] type = "websocket"  # or "stdio"
port = 9100

[permissions]
# Pre-grant permissions to skip approval prompts
bash = "granted"
edit = "granted"
write = "granted"

[tools]
# Conditional tool availability
websearch = false           # requires Exa API key
codesearch = false          # requires Exa API key
lsp = false                 # experimental
batch = true
apply_patch = "auto"        # enabled for GPT-4.1+ models automatically

[skills]
dirs = [".huston/skills", "~/.huston/skills"]
```

### 6.2 Model Switching at Runtime

Frontends can send `set_model` events to hot-swap the model mid-session:

```typescript
// Server handles set_model event
case "set_model":
  const newModel = createAdapter({
    provider: event.provider,
    model: event.model,
    stream: true,
  });
  agent.setModel(newModel);  // Glove's built-in hot-swap
  break;
```

This is critical for local model support — users can start with Claude for complex planning, then switch to a local model for routine edits.

---

## 7. Conditional Tool Availability

Like OpenCode, some tools are conditionally registered based on config, environment, and model:

```typescript
function createToolRegistry(cwd: string, store: SqliteStore, display: Displaymanager, config: HustonConfig): GloveFoldArgs<any>[] {
  const tools: GloveFoldArgs<any>[] = [
    // Always available
    createBashTool(cwd),
    createReadTool(cwd),
    createGlobTool(cwd),
    createGrepTool(cwd),
    createListTool(cwd),
    createWebfetchTool(),
    createSkillTool(config.skillDirs),
    createQuestionTool(),
  ];

  // Edit tools — either apply_patch OR (write + edit + multiedit)
  if (shouldUseApplyPatch(config.model)) {
    tools.push(createApplyPatchTool(cwd));
  } else {
    tools.push(createWriteTool(cwd));
    tools.push(createEditTool(cwd));
    tools.push(createMultieditTool(cwd));
  }

  // Conditional
  if (config.tools.websearch && process.env.EXA_API_KEY) {
    tools.push(createWebsearchTool());
  }
  if (config.tools.codesearch && process.env.EXA_API_KEY) {
    tools.push(createCodesearchTool());
  }
  if (config.tools.batch) {
    tools.push(createBatchTool(toolMap));
  }
  if (config.tools.lsp) {
    tools.push(createLspTool(cwd));
  }

  // Sub-agent tool
  tools.push(createTaskTool(cwd, display, config, agentRegistry));

  // Plan mode tools
  tools.push(createPlanEnterTool(sessionState));
  tools.push(createPlanExitTool(sessionState));

  // Wrap destructive tools with git snapshots
  return tools.map(t =>
    DESTRUCTIVE_TOOLS.has(t.name) ? withGitSnapshot(t, cwd) : t
  );
}
```

---

## 8. Persistent Shell Session

OpenCode maintains a persistent bash session across tool calls (environment variables, cd state, etc). Huston does the same using `node-pty` or a long-running `child_process`:

```typescript
class ShellSession {
  private process: ChildProcess;
  private cwd: string;

  constructor(cwd: string) {
    this.cwd = cwd;
    this.process = spawn("bash", ["--norc", "--noprofile", "-i"], {
      cwd,
      env: { ...process.env, TERM: "dumb" },
    });
  }

  async exec(command: string, timeout: number): Promise<ExecResult> {
    // Send command with unique delimiter markers
    // Capture output between markers
    // Return { stdout, stderr, exitCode }
  }

  destroy() {
    this.process.kill();
  }
}
```

One `ShellSession` per Huston session. Destroyed when the session ends.

---

## 9. Output Truncation

OpenCode truncates tool output at configurable line/byte limits and writes full output to a temp file readable via `read` with offset/limit. Huston follows the same pattern:

```typescript
interface TruncationResult {
  content: string;
  wasTruncated: boolean;
  totalLines: number;
  tempFilePath?: string;  // if truncated, full output written here
}

function truncateOutput(raw: string, opts: { maxLines: number; maxBytes: number }): TruncationResult {
  const lines = raw.split("\n");
  if (lines.length <= opts.maxLines && Buffer.byteLength(raw) <= opts.maxBytes) {
    return { content: raw, wasTruncated: false, totalLines: lines.length };
  }

  const truncated = lines.slice(0, opts.maxLines).join("\n");
  const tempPath = join(tmpdir(), `huston-output-${randomId()}.txt`);
  writeFileSync(tempPath, raw);

  return {
    content: truncated + `\n\n[Output truncated. Full output: ${tempPath} (${lines.length} lines)]`,
    wasTruncated: true,
    totalLines: lines.length,
    tempFilePath: tempPath,
  };
}
```

---

## 10. Phasing

### Phase 1 — Core Agent (this RFC)

- All tools from §3.1 (except LSP)
- Plan mode (`plan_enter` / `plan_exit` with read-only enforcement via runtime prompts)
- Session naming — auto-generated titles via title sub-agent + user rename
- Session summary generation via summary sub-agent
- SqliteStore persistence with session resume
- WebSocket + stdio transports
- Per-provider system prompts
- Context compaction
- Permission system
- Git safety snapshots
- Config file loading
- Sub-agent spawning (`task` tool) with display bubble-up
- Configurable sub-agent model override
- `batch` tool for parallel execution
- Output truncation

### Phase 2 — Extended Features

- LSP tool (experimental, opt-in)
- Custom agent generation from natural language
- MCP server support (expose Huston as an MCP tool provider)
- `apply_patch` auto-detection for compatible models
- OpenCode skill format interoperability (`.opencode/skills/` → `.huston/skills/` bridge)

### Phase 3 — Multi-Agent Platform

- Agent registry API for registering new agent types at runtime
- Inter-agent communication (not just parent → child)
- Shared context between agents
- Agent marketplace / plugin system

---

## 11. Directory Structure

```
huston/
├── src/
│   ├── index.ts              # Entry point — CLI or daemon
│   ├── config.ts             # Config loading (TOML)
│   ├── session.ts            # Session lifecycle (create, resume, list, destroy)
│   ├── session-state.ts      # Modal state (plan mode flag, step counter)
│   ├── agent.ts              # Glove agent construction
│   ├── prompt/
│   │   ├── anthropic.ts      # Anthropic system prompt
│   │   ├── beast.ts          # GPT-4.x / o1/o3 system prompt
│   │   ├── codex.ts          # GPT-5 system prompt
│   │   ├── gemini.ts         # Gemini system prompt
│   │   ├── default.ts        # Fallback (Qwen-derived, good for local)
│   │   ├── compaction.ts     # Compaction instructions
│   │   ├── environment.ts    # Dynamic environment block
│   │   └── runtime/
│   │       ├── plan-mode.ts
│   │       ├── build-switch.ts
│   │       └── max-steps.ts
│   ├── tools/
│   │   ├── index.ts          # Tool registry factory
│   │   ├── bash.ts
│   │   ├── read.ts
│   │   ├── write.ts
│   │   ├── edit.ts
│   │   ├── multiedit.ts
│   │   ├── apply-patch.ts
│   │   ├── glob.ts
│   │   ├── grep.ts
│   │   ├── list.ts
│   │   ├── webfetch.ts
│   │   ├── websearch.ts
│   │   ├── codesearch.ts
│   │   ├── question.ts
│   │   ├── task.ts           # Sub-agent spawning with display bubble-up
│   │   ├── batch.ts
│   │   ├── skill.ts
│   │   ├── plan-enter.ts
│   │   ├── plan-exit.ts
│   │   └── lsp.ts            # Phase 2
│   ├── agents/
│   │   ├── registry.ts       # Agent type registry
│   │   ├── explore.ts        # Read-only codebase navigator
│   │   ├── code.ts           # Full coding agent config
│   │   ├── plan.ts           # Planning agent config
│   │   ├── compaction.ts     # Compaction agent config
│   │   ├── summary.ts        # Session summary generator
│   │   └── title.ts          # Session title generator
│   ├── display/
│   │   └── proxy.ts          # DisplayProxy for sub-agent bubble-up
│   ├── transport/
│   │   ├── types.ts          # ServerEvent, ClientEvent contracts
│   │   ├── websocket.ts      # WebSocket transport
│   │   └── stdio.ts          # stdio transport
│   ├── shell.ts              # Persistent shell session (child_process + markers)
│   └── git.ts                # Git snapshot utility
├── skills/                   # Built-in skills (shipped with Huston)
├── package.json
└── tsconfig.json
```

---

## 12. Plan Mode

Plan mode is a modal state where the agent switches from a read-write coding agent to a read-only planning agent. This maps to OpenCode's `plan_enter` / `plan_exit` tools and the runtime-injected `PLAN_MODE_PROMPT`.

### 12.1 How It Works

Plan mode is implemented as a state flag on the session, enforced via runtime prompt injection:

```typescript
class SessionState {
  private _planMode = false;
  private agent: IGloveRunnable;
  private store: SqliteStore;

  async enterPlanMode() {
    this._planMode = true;
    // Inject read-only constraint as a system reminder
    await this.store.appendMessages([{
      sender: "user",
      text: `<system-reminder>${PLAN_MODE_PROMPT}</system-reminder>`,
    }]);
  }

  async exitPlanMode() {
    this._planMode = false;
    await this.store.appendMessages([{
      sender: "user",
      text: `<system-reminder>${BUILD_SWITCH_PROMPT}</system-reminder>`,
    }]);
  }

  get isPlanMode() { return this._planMode; }
}
```

The `plan_enter` and `plan_exit` tools trigger these state transitions:

```typescript
export function createPlanEnterTool(session: SessionState): GloveFoldArgs<{}> {
  return {
    name: "plan_enter",
    description: "Suggest switching to plan mode when a request would benefit from planning before implementation. Call when the request is complex, involves multiple files, or has significant architectural decisions.",
    inputSchema: z.object({}),
    async do(_input, display) {
      // Ask user for confirmation via pushAndWait (bubbles up to frontend)
      const confirmed = await display.pushAndWait({
        renderer: "confirm",
        input: {
          message: "The agent suggests switching to planning mode. This will restrict it to read-only until a plan is finalized. Switch?",
          options: ["Yes, plan first", "No, implement directly"],
        },
      });

      if (confirmed === "Yes, plan first") {
        await session.enterPlanMode();
        return { status: "success", data: "Switched to plan mode. Agent is now read-only." };
      }
      return { status: "success", data: "User chose to skip planning. Continuing with implementation." };
    },
  };
}

export function createPlanExitTool(session: SessionState): GloveFoldArgs<{}> {
  return {
    name: "plan_exit",
    description: "Signal that planning is complete. Call after writing the plan file and clarifying all open questions.",
    inputSchema: z.object({}),
    async do(_input, display) {
      const confirmed = await display.pushAndWait({
        renderer: "confirm",
        input: {
          message: "The agent has completed its plan. Switch to build mode?",
          options: ["Yes, start building", "No, continue planning"],
        },
      });

      if (confirmed === "Yes, start building") {
        await session.exitPlanMode();
        return { status: "success", data: "Switched to build mode. Agent can now modify files." };
      }
      return { status: "success", data: "Continuing in plan mode." };
    },
  };
}
```

### 12.2 Anthropic-Enhanced Planning Workflow

For Anthropic models, the plan mode system prompt includes a structured 5-phase workflow (matching OpenCode's `plan-reminder-anthropic.txt`):

1. **Initial Understanding** — launch up to 3 explore sub-agents in parallel via `task`
2. **Planning** — synthesize findings into a plan
3. **Synthesis** — ask user trade-off questions via `question`
4. **Final Plan** — write plan to `.huston/plans/<session-id>.md`
5. **Exit** — call `plan_exit`

### 12.3 Tool Gating in Plan Mode

Destructive tools are NOT de-registered in plan mode (that would require rebuilding the Glove instance). Instead, the runtime prompt strictly forbids their use, and an optional enforcement layer can intercept tool calls:

```typescript
// Optional hard enforcement — wraps destructive tools with a plan-mode check
function withPlanModeGuard<I>(tool: GloveFoldArgs<I>, session: SessionState): GloveFoldArgs<I> {
  const originalDo = tool.do;
  return {
    ...tool,
    async do(input, display) {
      if (session.isPlanMode) {
        return {
          status: "error",
          data: null,
          message: `Tool "${tool.name}" is blocked in plan mode. Use plan_exit to switch to build mode first.`,
        };
      }
      return originalDo(input, display);
    },
  };
}
```

---

## 13. Session Naming & Summary

### 13.1 Auto-Generated Titles

After the first user message + agent response, Huston generates a session title using a lightweight sub-agent. This runs in the background (non-blocking) and updates the store:

```typescript
async function generateSessionTitle(store: SqliteStore, config: HustonConfig) {
  const messages = await store.getMessages();
  if (messages.length < 2) return;  // need at least one exchange

  const titleModel = createAdapter({
    provider: config.provider,
    model: config.model,
    stream: false,
    maxTokens: 60,
  });

  const titleStore = new SqliteStore({ dbPath: config.dbPath, sessionId: `title-${randomId()}` });
  const titleDisplay = new Displaymanager();

  const titleAgent = new Glove({
    store: titleStore,
    model: titleModel,
    displayManager: titleDisplay,
    systemPrompt: TITLE_PROMPT,  // "Output ONLY a thread title. ≤50 chars. ..."
    compaction_config: { compaction_instructions: "", max_turns: 2 },
  }).build();

  // Feed the first exchange to the title agent
  const firstUser = messages.find(m => m.sender === "user");
  const firstAgent = messages.find(m => m.sender === "agent");
  const prompt = `User: ${firstUser?.text}\nAssistant: ${firstAgent?.text?.slice(0, 200)}`;

  await titleAgent.processRequest(prompt);
  const titleMessages = await titleStore.getMessages();
  const title = titleMessages.filter(m => m.sender === "agent").pop()?.text?.trim();

  if (title) {
    store.setName(title);
  }

  // Clean up the ephemeral title session
  titleStore.close();
}
```

The transport emits a `session_title` event so the frontend can update its sidebar/header:

```typescript
// Added to ServerEvent union
| { type: "session_title"; title: string }
```

### 13.2 Session Summary

On session end (or on demand), a summary sub-agent generates a PR-style description:

```typescript
async function generateSessionSummary(store: SqliteStore, config: HustonConfig): Promise<string> {
  // Similar pattern to title generation, but uses SUMMARY_PROMPT
  // "Summarize what was done. Write like a PR description. 2-3 sentences max. ..."
  // Returns the summary text
}
```

### 13.3 Frontend Session Management

The transport exposes session lifecycle operations:

```typescript
// Additional ClientEvent types
| { type: "session_list" }                           // list all sessions
| { type: "session_resume"; sessionId: string }      // resume a session
| { type: "session_rename"; title: string }           // manual rename
| { type: "session_delete"; sessionId: string }       // delete a session
| { type: "session_summarize" }                       // trigger summary generation

// Additional ServerEvent types
| { type: "session_list_result"; sessions: SessionInfo[] }
| { type: "session_title"; title: string }
| { type: "session_summary"; summary: string }
```

```typescript
interface SessionInfo {
  id: string;
  name: string | null;
  workingDir: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

## 14. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | `node-pty` vs `child_process`? | `child_process` with marker-based protocol | Simpler, more portable, matches OpenCode's approach. No native dep. |
| 2 | Transport auth? | No auth for Phase 1 | Single-user local-only is sufficient. Revisit for multi-user in Phase 3. |
| 3 | Sub-agent model selection? | Configurable via `modelOverride` per agent type | Explore agents can use cheaper models (Haiku). Code agents inherit parent's model. User can override in config. |
| 4 | Include `batch` tool? | Yes | Useful for models that don't emit parallel tool calls natively. Proven efficiency gain from OpenCode. |
| 5 | Skill directory? | `.huston/skills/` | Own convention for now. Interoperability with `.opencode/skills/` deferred to Phase 2. |

## 15. Open Questions

1. **Max steps enforcement:** OpenCode injects a `MAX_STEPS_PROMPT` when the agent hits a step limit, disabling tools. Should Huston implement this as a hard tool-call gate (reject all tool calls after N steps) or rely purely on the prompt injection? Hard gating is safer but less flexible.

2. **Plan file location:** OpenCode writes plans to a project-level file. Should Huston use `.huston/plans/<session-id>.md` (keeps plans organized per-session) or a single `.huston/plan.md` (simpler, matches the "current plan" mental model)?

3. **Sub-agent depth limit:** With the bubble-up proxy, sub-agents can spawn sub-agents recursively. Should there be a max depth (e.g. 3 levels) to prevent runaway nesting?

---

## 16. Dependencies

```json
{
  "dependencies": {
    "glove-core": "latest",
    "zod": "^3.23",
    "better-sqlite3": "^11.0",
    "fast-glob": "^3.3",
    "ws": "^8.17",
    "toml": "^3.0"
  },
  "devDependencies": {
    "typescript": "^5.5",
    "@types/better-sqlite3": "^7.6",
    "@types/ws": "^8.5"
  }
}
```

Note: `better-sqlite3` comes transitively via `glove-core` (for `SqliteStore`), but is listed explicitly since we depend on it directly for session management.

---

## Summary

Huston's coding agent is a server-side `glove-core` application that replicates OpenCode's full tool set and agent capabilities. By running headless with a transport layer, it decouples the intelligence from the interface. Any frontend — CLI, web, desktop — connects as a thin subscriber. Local model support comes free from Glove's `OpenAICompatAdapter`.

The key architectural contribution is the **display bubble-up** pattern: sub-agents proxy their `Displaymanager` through the parent, so `pushAndWait` slots (questions, permission prompts) surface on the frontend regardless of agent depth. This makes sub-agents first-class citizens that can interact with users without special frontend wiring.

Phase 1 delivers the complete coding agent: all tools, plan mode, session naming/summary, sub-agent spawning with bubble-up, and configurable model overrides per agent type. The architecture is designed so that coding is just the first agent type; the same infrastructure supports any future specialized agent.
