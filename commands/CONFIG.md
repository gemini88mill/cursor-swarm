# Swarm Manager Config Reference

Use `config.example.json` as a template. Copy it to `config.json` (or to `~/.cursor/agents/manager/config.json` if used by the swarm manager) and customize as needed.

## Top-level fields

| Field | Description | Example |
|-------|-------------|---------|
| `subagents` | List of available subagents and their definitions | See below |
| `handoff-to` | Map of agent name → next possible agents in the workflow | `{"planner": ["implementor"]}` |
| `handoff-notes` | Optional notes per agent for handoff decisions | Human-readable hints |
| `loop-control` | Limits reviewer→implementor retry cycles | Set `null` to disable |
| `plan-validation` | How often to allow replanning before escalating | `maxReplans: 1` |
| `parallelization` | Enable parallel task execution | `enabled: true` + rules |
| `behavior` | Planning and branching policies | `planning`, `branching` |
| `escalation-actions` | What to do when limits are hit | `needs-human`, etc. |

## subagents (each entry)

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✓ | Agent identifier; used as `subagent_type` in Task tool |
| `description` | ✓ | Short description for the manager |
| `invoke` | ✓ | How to invoke (for prompts) |
| `inputType` | ✓ | `"user-task"` (planner) or `"handoff"` |
| `first-step` | One only | `true` for the agent that starts the run |
| `schemaPath` | ✓ | Path to result schema (e.g. `~/.cursor/agents/...`) |
| `planPath` | optional | Path to agent definition .md file |

## handoff-to

Defines the workflow graph. Each key is an agent; each value is an array of agents it can hand off to.

**Example**: `"reviewer": ["implementor", "tester"]` → reviewer can either send back to implementor (reject) or to tester (approve).

## loop-control

Limits how many times the manager can send work from `fromAgent` back to `toAgent`.

| Field | Example | Description |
|-------|---------|-------------|
| `fromAgent` | `"reviewer"` | Agent that may reject |
| `toAgent` | `"implementor"` | Agent that receives the retry |
| `maxLoops` | `3` | Maximum reject→fix cycles |
| `countField` | `"review-fix-loop-count"` | Field in run.status.json to track count |
| `escalation-action` | `"needs-human"` | Outcome when limit exceeded |

Set to `null` or omit to disable.

## plan-validation

| Field | Example | Description |
|-------|---------|-------------|
| `maxReplans` | `1` | Times to request replanning before escalating |
| `escalation-action` | `"needs-human"` | Outcome when replan limit exceeded |

## parallelization

| Field | Example | Description |
|-------|---------|-------------|
| `enabled` | `true` | Allow parallel task batches |
| `rules` | Array | Per-rule limits |
| `rules[].maxConcurrent` | `4` | Max parallel workers |
| `rules[].requireDisjointFiles` | `true` | Only parallelize non-overlapping file sets |
| `rules[].respectDependencies` | `true` | Honor task dependencies |

## behavior

| Section | Field | Example | Description |
|---------|-------|---------|-------------|
| `planning` | `enabled` | `true` | Use plan validation flow |
| `planning` | `planResultField` | `"plan-path"` | Result field pointing to plan.json |
| `planning` | `planSchemaPath` | `"~/.cursor/agents/plan/plan.schema.json"` | Plan schema |
| `planning` | `replanCountField` | `"plan-replan-count"` | run.status field for replan count |
| `branching` | `enabled` | `true` | Use worktree per run, create PR at end |

## Paths

Schema and plan paths use `~` for the home directory (e.g. `~/.cursor/agents/plan/plan.schema.json`). Adjust for your system if needed.
