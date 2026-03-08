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

### Testing Strategy

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
4. Help the user set up credentials directly:
   - Create .env file with all needed variables
   - Test connections where possible (e.g. curl Chatwoot API)
   - Don't just list instructions — do the work
5. Present analysis summary

### PHASE 2: ARCHITECTURE & PLANNING

1. Invoke relevant integration skills in the main conversation:
   - If Chatwoot detected: invoke `chatwoot-skills:chatwoot-automation-patterns`
   - If other integrations: invoke their respective skills
   (The conversion-architect agent already has langgraph + conversion
   skills preloaded, but integration-specific skills should be invoked
   here so the main conversation also has the context)
2. Launch `conversion-architect` agent with:
   - Full consolidated analysis from Phase 1
   - List of credentials available in .env
   - User's language preference
3. Architect produces the conversion plan including:
   - Architecture overview
   - State schemas, node/edge definitions, tool specs
   - System prompts: ORIGINAL + PROPOSED ADAPTATION + DIFF
   - Package list (exact names for `bun add` — NEVER manual package.json)
   - Milestone-based test plan
   - Implementation order as phased checklist
   - Graph visualization script spec (visualize-graphs.ts)
4. Present plan summary to user for review

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
   - Agent C focus: Integration correctness + error handling
   (All reviewers have langgraph + n8n + conversion skills preloaded)
2. Consolidate findings
3. Present review report:
   - Fidelity score per workflow
   - Critical issues (confidence >= 90%)
   - Important issues (confidence >= 80%)
   - Low-confidence observations (< 80%, for documentation)
   - Improvement suggestions
4. Ask user which issues to address
