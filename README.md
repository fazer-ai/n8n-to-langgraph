# n8n-to-langgraph

A Claude Code marketplace plugin that converts n8n workflows to LangGraph TypeScript applications.

## What it does

This plugin provides a structured pipeline for converting n8n workflow JSON files into fully functional LangGraph applications. It handles the full lifecycle: analysis, architecture planning, implementation guidance, and post-implementation review.

## Architecture

3 specialized subagents + 1 orchestrator skill:

```
/n8n-to-langgraph <workflows-dir>
│
├─► workflow-analyzer (×2-3 in parallel)
│   Parses n8n JSON, extracts system prompts verbatim,
│   traces execution paths, maps integrations & credentials
│
├─► conversion-architect
│   Designs LangGraph StateGraph topology, state schemas,
│   tool specs, test plan, and phased implementation order
│
└─► conversion-reviewer (review mode)
    Compares implementation against originals for fidelity,
    test coverage, and code quality
```

Each agent has domain-specific skills preloaded via frontmatter — they don't inherit from the parent conversation. This guarantees the right knowledge is available regardless of how the agent is invoked.

## Plugin structure

```
.claude-plugin/
├── plugin.json              # Plugin metadata
└── marketplace.json         # Marketplace registration
agents/
├── workflow-analyzer.md     # n8n workflow analysis agent
├── conversion-architect.md  # LangGraph architecture design agent
└── conversion-reviewer.md   # Post-implementation review agent
skills/
└── n8n-to-langgraph/
    └── SKILL.md             # Orchestrator skill + domain knowledge
```

## Installation

Add to your `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "n8n-to-langgraph@n8n-to-langgraph": true
  },
  "extraKnownMarketplaces": {
    "n8n-to-langgraph": {
      "source": {
        "source": "git",
        "url": "https://github.com/fazer-ai/n8n-to-langgraph.git"
      }
    }
  }
}
```

Then restart Claude Code.

## Usage

### Convert workflows

```
/n8n-to-langgraph workflows/
```

This runs two phases:

1. **Analysis** — Parallel agents parse your n8n workflow JSONs, extract every system prompt verbatim, trace execution paths, and identify all integrations and credentials needed.

2. **Architecture** — An architect agent produces a detailed conversion plan: state schemas, tool specs, graph topology, original vs adapted prompts with diffs, milestone-based test plan, and phased implementation checklist.

The plan is then handed off to [ralph-loop](https://github.com/anthropics/claude-plugins-official) for autonomous implementation.

### Review a conversion

```
/n8n-to-langgraph review
```

Launches parallel reviewers that compare the implementation against the original workflows, checking prompt fidelity, logic completeness, test coverage, and integration correctness. Issues are confidence-scored (0-100) so you can focus on what matters.

## Concept mapping

| n8n | LangGraph |
|---|---|
| Workflow | StateGraph |
| Sub-workflow | Tool or nested graph |
| Webhook trigger | HTTP route |
| AI Agent node | createReactAgent |
| Code node | Utility function or graph node |
| IF/Switch node | Conditional edges |
| Credentials | Environment variables (.env) |
| Execution data | State schema (Annotation) |

## Dependencies

This plugin works best with these companion plugins:

- [n8n-mcp-skills](https://github.com/czlonkowski/n8n-skills) — n8n domain knowledge for the analyzer agents
- [chatwoot-skills](https://github.com/fazer-ai/chatwoot-skills) — Chatwoot integration knowledge (if your workflows use Chatwoot)
- [ralph-loop](https://github.com/anthropics/claude-plugins-official) — Autonomous implementation loop

LangGraph skills (`langgraph-fundamentals`, `langgraph-persistence`, `langchain-dependencies`) are expected to be available in your Claude Code environment.

## License

MIT
