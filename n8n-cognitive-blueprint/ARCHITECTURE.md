# Cognitive Blueprint Runtime Agent - n8n Implementation

## Source Framework

Based on the [MarkTechPost tutorial: Building Next-Gen Agentic AI](https://www.marktechpost.com/2026/03/07/building-next-gen-agentic-ai-a-complete-framework-for-cognitive-blueprint-driven-runtime-agents-with-memory-tools-and-validation/) which presents a Python framework for cognitive blueprint-driven runtime agents.

## Architecture Overview

The framework implements a **Plan-Execute-Validate** loop driven by declarative cognitive blueprints. Each agent's behavior is defined by a YAML blueprint specifying identity, goals, constraints, tools, memory configuration, planning strategy, and validation rules.

```
┌─────────────────────────────────────────────────────────────────┐
│                    n8n Workflow Architecture                     │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ Webhook  │──▶│    Load      │──▶│    Tool      │            │
│  │ Trigger  │   │  Blueprint   │   │  Registry    │            │
│  └──────────┘   └──────────────┘   └──────┬───────┘            │
│                                           │                     │
│                                           ▼                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              PHASE 1: PLANNING                         │     │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────┐   │     │
│  │  │  Build     │──▶│  Planner   │──▶│  Parse       │   │     │
│  │  │  Prompt    │   │  LLM Call  │   │  Plan        │   │     │
│  │  └────────────┘   └────────────┘   └──────────────┘   │     │
│  └────────────────────────────────────────┬───────────────┘     │
│                                           │                     │
│                                           ▼                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              PHASE 2: EXECUTION                        │     │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────┐   │     │
│  │  │  Execute   │──▶│  Needs LLM │──▶│  Reasoning   │   │     │
│  │  │  Step      │   │  Reasoning?│   │  LLM Call    │   │     │
│  │  └────────────┘   └─────┬──────┘   └──────┬───────┘   │     │
│  │       ▲                 │                  │           │     │
│  │       │                 ▼                  │           │     │
│  │       │          ┌──────────────┐          │           │     │
│  │       └──────────│  More Steps? │◀─────────┘           │     │
│  │                  └──────────────┘                      │     │
│  └────────────────────────────────────────┬───────────────┘     │
│                                           │                     │
│                                           ▼                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              PHASE 3: SYNTHESIS                        │     │
│  │  ┌────────────┐   ┌────────────┐                      │     │
│  │  │  Build     │──▶│  Synthesis │                      │     │
│  │  │  Synthesis │   │  LLM Call  │                      │     │
│  │  └────────────┘   └────────────┘                      │     │
│  └────────────────────────────────────────┬───────────────┘     │
│                                           │                     │
│                                           ▼                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │              PHASE 4: VALIDATION                       │     │
│  │  ┌────────────┐   ┌────────────┐   ┌──────────────┐   │     │
│  │  │ Validation │──▶│ Validation │──▶│   Retry /    │   │     │
│  │  │  Engine    │   │  Passed?   │   │   Format     │   │     │
│  │  └────────────┘   └────────────┘   └──────────────┘   │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Component Mapping: Python Framework → n8n Nodes

| Python Component | n8n Node(s) | Purpose |
|---|---|---|
| `CognitiveBlueprint` (Pydantic model) | **Load Cognitive Blueprint** (Code) | Loads and validates blueprint configuration |
| `ToolRegistry` + `ToolSpec` | **Tool Registry** (Code) | Registers tools and builds descriptions |
| `Planner._build_planner_prompt()` | **Build Planner Prompt** (Code) | Constructs LLM prompt from blueprint |
| `Planner.plan()` | **Planner LLM Call** (Anthropic, Claude Sonnet 4.6) | Generates structured execution plan |
| `Plan` + `PlanStep` dataclasses | **Parse Plan** (Code) | Parses LLM JSON into plan steps |
| `Executor.execute_plan()` | **Execute Step** (Code) | Routes steps to tools or reasoning |
| `ToolRegistry.call()` | **Execute Step** (Code, switch block) | Executes tool functions inline |
| Executor reasoning branch | **Reasoning LLM Call** (Anthropic, Claude Haiku 4.5) | LLM-based reasoning for non-tool steps |
| `Executor._synthesize()` | **Synthesis LLM Call** (Anthropic, Claude Sonnet 4.6) | Combines step results into final answer |
| `Validator.validate()` | **Validation Engine** (Code) | Checks answer against blueprint rules |
| `MemoryManager` | **Validation Engine** (Code, memory section) | Tracks conversation history |
| `RuntimeAgent.run()` | Entire workflow orchestration | Plan → Execute → Validate loop |
| `RuntimeAgent._display_answer()` | **Format Response** (Code) | Structures output for API response |

## Cognitive Blueprint Schema

Each blueprint defines the complete cognitive profile of an agent:

```yaml
identity:           # BlueprintIdentity - who the agent is
  name: string
  version: string
  description: string
  author: string

goals: [string]     # What the agent tries to achieve
constraints: [string] # What the agent must not do
tools: [string]     # Which registered tools the agent can use

memory:             # BlueprintMemory - conversation memory config
  type: short_term | episodic | persistent
  window_size: int          # Recent messages to keep
  summarize_after: int      # Compress after N messages

planning:           # BlueprintPlanning - plan generation config
  strategy: sequential | hierarchical | reactive
  max_steps: int
  max_retries: int
  think_before_acting: bool

validation:         # BlueprintValidation - output validation rules
  require_reasoning: bool
  min_response_length: int
  forbidden_phrases: [string]
```

## Available Tools

| Tool | Parameters | Description |
|---|---|---|
| `calculator` | `expression` | Safe mathematical expression evaluation |
| `unit_converter` | `value`, `from_unit`, `to_unit` | Unit conversion (km/miles, kg/lbs, etc.) |
| `date_calculator` | `operation`, `date1`, `date2` | Date arithmetic and difference |
| `search_knowledge_base` | `topic` | Knowledge base lookup (stub in demo) |
| `statistics_engine` | `numbers` | Descriptive statistics (mean, median, std_dev, etc.) |
| `list_sorter` | `numbers`, `order` | Sort numbers ascending or descending |

## Execution Flow

1. **Webhook receives** `{ "task": "...", "blueprint": "research" }`
2. **Load Blueprint** selects and validates the cognitive blueprint
3. **Tool Registry** registers tools and builds descriptions for the LLM
4. **Planner** generates a structured JSON execution plan via LLM
5. **Executor** iterates through plan steps:
   - Tool steps: executes the tool function directly
   - Reasoning steps: calls LLM for pure reasoning
   - Loops back until all steps complete
6. **Synthesis** combines all step outputs into a final answer via LLM
7. **Validation** checks the answer against blueprint rules
   - If failed: retry with feedback (up to `max_retries`)
   - If passed: format and return response
8. **Response** returns structured JSON with plan, results, answer, and validation

## Usage

### Import into n8n

1. Open n8n and go to **Workflows** → **Import from File**
2. Select `cognitive-blueprint-agent.workflow.json`
3. Configure your Anthropic API credentials (replace `ANTHROPIC_CREDENTIAL_ID`)
4. Activate the workflow

### LLM Model Configuration

The workflow uses a tiered Anthropic Claude model strategy optimized for accuracy vs. economy:

| LLM Call | Model | Rationale |
|---|---|---|
| **Planner** | Claude Sonnet 4.6 | Highest-leverage call — a bad plan wastes all downstream tokens. Strong structured JSON output. |
| **Reasoning** | Claude Haiku 4.5 | Fast, cheap, handles isolated reasoning subtasks well. Fires per-step so volume savings matter most. |
| **Synthesis** | Claude Sonnet 4.6 | User-facing final output. Worth Sonnet-tier quality for polished, accurate answers. |

### API Call

```bash
curl -X POST http://localhost:5678/webhook/cognitive-agent \
  -H "Content-Type: application/json" \
  -d '{
    "task": "If the Eiffel Tower is 330 meters tall, how many 20cm steps to climb it? What is the calorie burn at 0.15 cal/step?",
    "blueprint": "research"
  }'
```

### Response Structure

```json
{
  "agent": { "name": "ResearchBot", "version": "1.2.0", "strategy": "sequential" },
  "task": "...",
  "plan": { "total_steps": 4, "steps": [...] },
  "execution": { "results": [...] },
  "final_answer": "...",
  "validation": { "passed": true, "score": 1.0, "issues": [] },
  "memory": { "messages_stored": 2, "has_summary": false }
}
```

## Extending the Framework

### Adding a New Tool

Add to the `TOOL_REGISTRY` object in the **Tool Registry** node and add the execution case to the **Execute Step** node's switch block:

```javascript
// In Tool Registry node
custom_tool: {
  name: 'custom_tool',
  description: 'Does something custom',
  parameters: { input: 'Input description' },
  returns: 'Output description'
}

// In Execute Step node
case 'custom_tool': {
  result.output = doSomething(args.input);
  result.success = true;
  break;
}
```

### Adding a New Blueprint

Add a new YAML file in `blueprints/` and register it in the `BLUEPRINTS` object in the **Load Cognitive Blueprint** node.

### Connecting External Services

Replace stub tools with n8n HTTP Request nodes or dedicated integration nodes (e.g., connect `search_knowledge_base` to a real vector database or search API).
