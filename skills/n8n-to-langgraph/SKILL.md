---
name: n8n-to-langgraph
description: "INVOKE THIS SKILL when converting n8n workflows to LangGraph applications, analyzing n8n workflow JSON files for conversion, or reviewing an n8n-to-LangGraph conversion. Covers the full pipeline: workflow analysis, architecture planning, implementation via ralph-loop, and review."
argument-hint: "<workflows-dir> [review]"
---

# n8n → LangGraph Conversion

## Conversion Domain Knowledge

### Concept Mapping

| n8n Concept | LangGraph Equivalent |
|---|---|
| Workflow | StateGraph |
| Sub-workflow | Tool (StructuredTool) or nested graph |
| Webhook trigger | HTTP route (ElysiaJS/Express/etc.) |
| Schedule trigger | Cron job or external scheduler |
| AI Agent node | createReactAgent (inner agent) |
| Code node | Utility function or graph node |
| IF/Switch node | Conditional edges |
| Set/Edit Fields | State update in node return |
| Sticky Notes | Code comments |
| Credentials | Environment variables (.env) |
| Execution data | State schema (Annotation) |
| Wait node | sleep() in graph node |
| Error trigger | try/catch + conditional edge |
| — (no n8n equivalent) | Langfuse observability (CallbackHandler) |
| — (no n8n equivalent) | Logger service (structured logging) |

### Prompt Extraction

Where to find prompts in n8n JSON:
- `parameters.systemMessage` — AI Agent system prompt
- `parameters.text` — AI Agent user message template
- `parameters.messages` — Chat model message array
- AI Agent tool descriptions — in connected tool nodes' `parameters.description`
- Code nodes may contain prompt templates as string literals

### State Management

n8n uses per-execution data that flows between nodes. LangGraph uses persistent
checkpointers (PostgresSaver, MemorySaver) with thread-based state.

Key differences:
- n8n: data is ephemeral per execution, stored in `$json`, `$node[]`
- LangGraph: state persists across invocations via checkpointer
- Map n8n execution data → LangGraph state schema (Annotation.Root)
- Map n8n static data → LangGraph Store or database

### HTTP Server Binding

Generated apps MUST bind to `0.0.0.0` (all interfaces), NOT `localhost` or
`127.0.0.1`. This ensures the server is reachable from external services
(e.g., Chatwoot webhooks).

```typescript
// ElysiaJS
app.listen({ hostname: "0.0.0.0", port: Number(process.env.PORT ?? 3000) });

// Express
app.listen(Number(process.env.PORT ?? 3000), "0.0.0.0");

// Bun.serve
Bun.serve({ hostname: "0.0.0.0", port: Number(process.env.PORT ?? 3000), fetch: handler });
```

### package.json Scripts

The generated `package.json` MUST include a `dev` script using Bun's `--hot` flag
for hot reloading (preserves state, faster than `--watch`):

```json
{
  "scripts": {
    "dev": "bun --hot src/index.ts",
    "start": "bun src/index.ts"
  }
}
```

`--hot` reloads modules in-place without restarting the process, keeping
HTTP connections and in-memory state alive. Use `--watch` only if `--hot`
causes issues with a specific library.

### Tool Factory Pattern

When tools need webhook/request context (e.g., conversation ID, phone number),
use factory functions that capture context in closure:

```typescript
function createTools(context: { telefone: string; conversationId: number }) {
  return [
    tool(async (input) => {
      // context.telefone available here
    }, { name: "...", schema: z.object({...}) })
  ];
}
```

### Logger Service (Default)

All converted LangGraph applications MUST include a centralized logger service
for structured, leveled logging. Use **pino** as the logging library.

**Package**: `pino` + `pino-pretty` (dev dependency)

**Helper module** — create `src/lib/logger.ts`:

```typescript
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  transport:
    process.env.NODE_ENV !== "production"
      ? { target: "pino-pretty", options: { colorize: true } }
      : undefined,
});

export function createChildLogger(context: Record<string, unknown>) {
  return logger.child(context);
}
```

Design principles:
- **Leveled**: `trace`, `debug`, `info`, `warn`, `error`, `fatal` — configurable via LOG_LEVEL env var
- **Structured**: all log entries are JSON in production, pretty-printed in dev
- **Child loggers**: use `createChildLogger` to add context (request ID, conversation ID, etc.)
- **Zero-cost in prod**: pino is fast and low-overhead by design

**Where to log** (be generous — logs are cheap, debugging without them is expensive):

| Location | What to log | Level |
|---|---|---|
| Server startup | Host, port, public IP, environment | `info` |
| Webhook received | Route, method, conversation ID, payload size | `info` |
| Graph invocation start | Graph name, thread ID, input summary | `info` |
| Graph invocation end | Graph name, thread ID, duration, output summary | `info` |
| Node entry/exit | Node name, state snapshot (key fields) | `debug` |
| Tool execution | Tool name, input params, output summary, duration | `debug` |
| LLM call | Model, token count, latency (via Langfuse, but also log) | `debug` |
| Conditional routing | Decision made, which branch taken, why | `debug` |
| External API call | Service, method, URL, status code, duration | `info` |
| External API error | Service, method, URL, status code, error body | `error` |
| State update | Key fields changed, new values (redact sensitive data) | `debug` |
| Error caught | Error message, stack trace, context | `error` |
| Webhook response sent | Status code, duration from receipt | `info` |
| Langfuse flush | Success/failure of handler shutdown | `debug` |
| Credential validation | Service name, success/failure (never log secrets) | `info` |

**Usage patterns**:

```typescript
import { logger, createChildLogger } from "./lib/logger";

// At server startup
logger.info({ host: "0.0.0.0", port: 3000, publicIp }, "Server started");

// In webhook handler — create child logger with request context
const log = createChildLogger({ route: "/webhook/chatwoot", conversationId });
log.info({ payloadSize: body.length }, "Webhook received");

// In graph node
log.debug({ node: "classify_intent", threadId }, "Entering node");
// ... node logic ...
log.debug({ node: "classify_intent", result: intent }, "Exiting node");

// In tool execution
log.debug({ tool: "buscar_agenda", input: { date } }, "Tool called");
const result = await fetchCalendar(date);
log.debug({ tool: "buscar_agenda", slots: result.length }, "Tool completed");

// Error handling
try {
  await chatwootApi.sendMessage(conversationId, text);
  log.info({ service: "chatwoot", conversationId }, "Message sent");
} catch (err) {
  log.error({ service: "chatwoot", conversationId, err }, "Failed to send message");
  throw err;
}
```

**CRITICAL**: Never log secrets, API keys, tokens, or full request bodies containing
sensitive user data. Redact or omit sensitive fields.

### Langfuse Observability (Default)

All converted LangGraph applications MUST include Langfuse integration by default
for tracing, token usage, latency, and cost monitoring.

**Packages**: `langfuse` + `langfuse-langchain`

**Environment variables** (add to .env alongside other credentials):
```env
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_BASEURL=https://cloud.langfuse.com
```

**Helper module** — create `src/lib/langfuse.ts` (or equivalent path):

```typescript
import { CallbackHandler } from "langfuse-langchain";

const langfuseActive =
  !!process.env.LANGFUSE_SECRET_KEY && !!process.env.LANGFUSE_PUBLIC_KEY;

export function createLangfuseHandler(
  traceName: string,
  opts: {
    sessionId?: string;
    userId?: string;
    metadata?: Record<string, unknown>;
    tags?: string[];
  } = {},
): CallbackHandler | undefined {
  if (!langfuseActive) return undefined;

  return new CallbackHandler({
    secretKey: process.env.LANGFUSE_SECRET_KEY!,
    publicKey: process.env.LANGFUSE_PUBLIC_KEY!,
    baseUrl: process.env.LANGFUSE_BASEURL ?? "https://cloud.langfuse.com",
    sessionId: opts.sessionId,
    userId: opts.userId,
    metadata: { ...opts.metadata, traceName },
    tags: opts.tags,
  });
}

export async function flushLangfuseHandler(
  handler: CallbackHandler | undefined,
): Promise<void> {
  if (handler) {
    await handler.shutdownAsync();
  }
}
```

Design principles:
- **Opt-in via env vars**: returns `undefined` when keys are missing — zero overhead in dev/test
- **One handler per trace**: each request/invocation gets its own handler so traces don't mix
- **`traceName` in metadata**: stored in metadata for dashboard filtering
- **Always call `shutdownAsync()`**: use try/finally to guarantee buffered events are flushed

**Usage in graph/agent invocations**:

```typescript
// For compiled graph.invoke()
const langfuseHandler = createLangfuseHandler("my-graph", {
  sessionId: threadId,
});
try {
  const result = await graph.invoke(initialState, {
    configurable: { thread_id: threadId },
    callbacks: langfuseHandler ? [langfuseHandler] : undefined,
  });
} finally {
  await flushLangfuseHandler(langfuseHandler);
}

// For createReactAgent .invoke()
const langfuseHandler = createLangfuseHandler("react-agent", {
  sessionId: conversationId,
  userId: userId,
});
try {
  const result = await agent.invoke(
    { messages },
    langfuseHandler ? { callbacks: [langfuseHandler] } : undefined,
  );
} finally {
  await flushLangfuseHandler(langfuseHandler);
}
```

Callback propagation is automatic — passing callbacks to `graph.invoke()` or
`agent.invoke()` propagates to all nested LLM calls, tool calls, and sub-chains.
No changes needed in tools or prompts.

### Testing Strategy

Use **Bun's native test runner** (`bun test`) — NEVER install vitest, jest, mocha,
or any external test framework. Bun has built-in `describe`, `it`, `expect`, `mock`,
`spyOn`, `beforeEach`, `afterEach`, and lifecycle hooks.

```typescript
import { describe, it, expect, mock, spyOn, beforeEach } from "bun:test";

describe("myTool", () => {
  it("should return expected result", async () => {
    const mockFetch = mock(() =>
      Promise.resolve(new Response(JSON.stringify({ ok: true })))
    );
    // ... test logic
    expect(mockFetch).toHaveBeenCalledTimes(1);
  });
});
```

Key Bun test features to use:
- `mock()` for creating mock functions (replaces `vi.fn()` / `jest.fn()`)
- `spyOn(object, method)` for spying on existing methods
- `mock.module("module-name", () => ...)` for module mocking
- Test files: `*.test.ts` naming convention
- Run with: `bun test` (auto-discovers test files)
- Run specific: `bun test src/tools/my-tool.test.ts`

Tests after each implementation milestone (not TDD):
1. **Foundation** (env, DB): validate config loading, connection setup
2. **Services** (API clients): mock HTTP, verify request shapes
3. **Tools**: each tool with mocked services, verify output
4. **Graphs**: routing logic with mock LLM, verify node transitions
5. **Routes**: webhook handlers end-to-end with mocked graph

### Graph Visualization

Create a `visualize-graphs.ts` script using LangGraph's built-in method:

```typescript
import { graph } from "./graphs/main-agent/graph";
const png = await graph.getGraph().drawMermaidPng();
await Bun.write("main-agent.png", png);
```

Run after implementation to verify graph topology visually.

### Common Pitfalls

1. **Simplified prompts** — Extract VERBATIM, adapt minimally
2. **Missing branches** — n8n IF/Switch can have many outputs; map ALL
3. **Lost error handling** — n8n error triggers → try/catch + conditional edges
4. **Credential gaps** — List ALL env vars needed before implementing
5. **Package.json edits** — NEVER edit manually; always `bun add <pkg>`
6. **Language mismatch** — Use user's language in code naming, not English
7. **Localhost binding** — Server MUST bind to `0.0.0.0`, never localhost/127.0.0.1
8. **Webhook URLs with localhost** — Always detect public IP (`curl -s ifconfig.me`)
   and use it for webhook URLs so external services can reach the app
9. **Manual webhook setup** — Use MCP Chatwoot tools to register webhooks
   automatically instead of asking the user to do it manually
10. **Missing logging** — Every graph node, tool, API call, webhook handler,
    and error path MUST have logging. Use the logger service, not console.log

---

## Orchestration Instructions

When the user invokes `/n8n-to-langgraph` (or context matches):

If $ARGUMENTS contains "review", go to **Review Mode** below.
Otherwise, run the Analysis + Planning pipeline:

### PHASE 1: WORKFLOW ANALYSIS

1. Identify all workflow JSON files in the `$ARGUMENTS` directory
2. Launch 2-3 `workflow-analyzer` agents in parallel
   - Each receives a batch of workflow file paths
   - Agents have n8n skills preloaded (n8n-workflow-patterns,
     n8n-node-configuration, n8n-code-javascript)
3. Read agent results and consolidate:
   - All integrations found (Chatwoot, Google Calendar, OpenAI, etc.)
   - All credentials/env vars needed
   - All system prompts extracted verbatim
   - Data flow between workflows
4. Present analysis summary to user
5. **CREDENTIAL SETUP (MANDATORY GATE — do not proceed to Phase 2 without this)**:
   - Create .env file with all needed variables (placeholders for values)
   - Include LANGFUSE_SECRET_KEY, LANGFUSE_PUBLIC_KEY, LANGFUSE_BASEURL in .env
   - Ask user to fill in the actual credentials
   - Test connections where possible (e.g. curl Chatwoot API with provided token)
   - Do not proceed until user confirms credentials are set up
   - If user wants to skip, acknowledge that manual testing won't be
     possible until credentials are configured later
6. **PUBLIC IP DETECTION & WEBHOOK SETUP**:
   - Detect the machine's public IP by running: `curl -s ifconfig.me`
   - Store the public IP for use in webhook URLs
   - Determine the app's PORT from .env (default 3000)
   - Build the webhook base URL: `http://<PUBLIC_IP>:<PORT>`
   - If Chatwoot is among the detected integrations, use MCP Chatwoot
     tools to configure the webhook:
     a. Use `mcp_chatwoot_list_inboxes` to find the target inbox
     b. Use `mcp_chatwoot_update_inbox` (or the appropriate MCP tool)
        to set the webhook URL to `http://<PUBLIC_IP>:<PORT>/webhook/chatwoot`
     c. Verify the webhook was registered by listing inbox details
   - If other services need webhook registration, document the URL
     for the user to configure manually
   - Save PUBLIC_IP and WEBHOOK_BASE_URL to .env for reference

### PHASE 2: ARCHITECTURE & PLANNING

1. Launch `conversion-architect` agent with:
   - Full consolidated analysis from Phase 1
   - List of credentials available in .env
   - User's language preference
   - Instruction to include Logger service module (`src/lib/logger.ts`) with
     structured logging throughout all layers (routes, nodes, tools, services)
   - Instruction to include Langfuse observability module (`src/lib/langfuse.ts`)
     and wire it into all graph/agent `.invoke()` calls
   - List of detected integrations (e.g. Chatwoot, Google Calendar, etc.)
     and their corresponding skills to invoke during architecture design
     (e.g. "If you need Chatwoot patterns, invoke chatwoot-skills:chatwoot-automation-patterns")
2. Architect produces the conversion plan including:
   - Architecture overview
   - State schemas, node/edge definitions, tool specs
   - Logger service spec (`src/lib/logger.ts`) and logging placement guide
   - Langfuse helper module spec (`src/lib/langfuse.ts`) and wiring into all invoke calls
   - System prompts: ORIGINAL + PROPOSED ADAPTATION + DIFF
   - Package list (exact names for `bun add` — NEVER manual package.json)
     — must include `langfuse`, `langfuse-langchain`, `pino`, and `pino-pretty` (dev)
   - Milestone-based test plan
   - Implementation order as phased checklist
   - Graph visualization script spec (visualize-graphs.ts)
3. Present plan summary to user for review

### RALPH-LOOP HANDOFF

When the user is satisfied with the plan, instruct them to run
(in the user's language, adapting the text below accordingly):

```
/ralph-loop:ralph-loop "Leia e siga o plano de conversão em
<PLAN_FILE_PATH>. Siga as diretrizes de implementação na
seção final do plano." --max-iterations 15 --completion-promise
'Todos os workflows convertidos, testes passando, verificação manual completa'
```

The implementation guidelines are embedded in the plan itself (written
by the conversion-architect), so the ralph-loop prompt only needs to
reference the plan file path. All rules — bun add, user's language,
skill invocation, testing, commits, etc. — live in the plan.

---

## Review Mode

When `$ARGUMENTS` contains "review":

1. Launch 2-3 `conversion-reviewer` agents in parallel:
   - Agent A focus: Prompt fidelity + logic completeness
   - Agent B focus: Test coverage + code quality
   - Agent C focus: Integration correctness + error handling + Langfuse wiring
   (All reviewers have langgraph + n8n + conversion skills preloaded)
2. Consolidate findings
3. Present review report:
   - Fidelity score per workflow
   - Critical issues (confidence >= 90%)
   - Important issues (confidence >= 80%)
   - Low-confidence observations (< 80%, for documentation)
   - Improvement suggestions
4. Ask user which issues to address
