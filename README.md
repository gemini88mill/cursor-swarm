# Swarm: Multi-Agent Orchestration

Swarm is a Cursor slash command that orchestrates multiple subagents to plan, implement, review, and test code changes. It uses a config-driven workflow and produces a PR when the run completes successfully.

---

## What Is Swarm?

Swarm runs as the **Manager**—a slash command that delegates work to specialized subagents instead of doing the work itself. Each subagent has a single responsibility and produces structured JSON output. The Manager coordinates handoffs, validates results against schemas, and enforces policies (clean tree, standards, loop limits).

**Key behaviors (when enabled in config):**
- **Worktree per run**: Each run uses an isolated git worktree; your main branch stays untouched.
- **PR on success**: At the end of a successful run, a branch is pushed and a PR is created.
- **Standards-first**: `.swarm/standards.md` defines project conventions; all agents must follow it.

---

## The Agents

| Agent        | Role            | Input       | Output                    |
|-------------|-----------------|-------------|----------------------------|
| **Planner** | Creates implementation plan | Task string or MD path | `plan.json`, `planner.result.json` |
| **Implementor** | Implements code from plan | `handoff.json` | `implementor.result.json` |
| **Reviewer** | Reviews changes against standards | `handoff.json` | `reviewer.result.json` |
| **Tester**  | Runs project tests        | `handoff.json` | `tester.result.json` |

### Planner
- **Only plans**; never writes code.
- Reads `.swarm/standards.md` and any feature docs.
- Writes `.swarm/runs/{id}/plan.json` with goal, acceptance criteria, lanes, tasks.
- Output must conform to `plan.schema.json`.

### Implementor
- Implements tasks from the handoff.
- **Only touches files in `context.allowedFiles`** (guard rail).
- Runs lint/typecheck before completing.
- When branching is enabled, works in the worktree directory.

### Reviewer
- **Only reviews**; never modifies code.
- Compares changes to `.swarm/standards.md`.
- Verdict: `approved` or `rejected` with `issues` list.
- If rejected, work goes back to the Implementor (up to `loop-control.maxLoops`).

### Tester
- **Only runs tests**; no code changes.
- Reads `.swarm/standards.md` for tool constraints (e.g. Bun vs npm).
- Runs the project test command and reports pass/fail.
- Last step in the workflow; no further handoff.

---

## Workflow

```
User task → Planner → [Plan validation] → Implementor → Reviewer
                                                ↑         ↓
                                                └─────────┴─→ Tester → PR (if passed)
```

1. User runs `/swarm <task>`.
2. Manager enforces clean working tree, creates `.swarm/runs/{run-id}/`.
3. **Planner** runs first; produces `plan.json` and `planner.result.json`.
4. Manager validates the plan; may reject and request replanning (up to `maxReplans`).
5. If branching enabled: Manager creates git worktree for branch `swarm/<friendly-name>`.
6. **Implementor** receives handoff; implements tasks in the worktree.
7. **Reviewer** inspects changes via `git diff`; approves or rejects.
8. If rejected: hand off back to Implementor (up to `maxLoops`).
9. If approved: hand off to **Tester**.
10. **Tester** runs the test suite.
11. If tests pass: Manager commits, pushes branch, creates PR, removes worktree.

---

## How to Invoke

### Run a task
In Cursor chat:
```
/swarm Add a user authentication page with login form
```
or
```
/swarm Implement the feature described in docs/features/auth.md
```

The text after `/swarm` is the task. It can be a description or a path to an MD file.

### Init-only (setup without running)
```
/swarm init
/swarm setup only
/swarm just init
/swarm no run
```

This will:
- Create `.swarm/` and `.swarm/runs/` if missing
- Check for `.swarm/standards.md`
- **Stop** without invoking any subagents

If standards are missing or placeholder, Swarm will ask you to create them.

---

## Prerequisites

1. **`.swarm/standards.md`**  
   Human-authored coding standards (style, patterns, tooling). Required before subagent execution. Swarm will not create this for you.

2. **Clean git working tree**  
   No modified or staged files. Untracked files are allowed. Commit or stash changes before running.

3. **Config**  
   Manager config at `~/.cursor/agents/manager/config.json` (or `~/.cursor/commands/config.json` depending on setup). Use `config.example.json` as a template.

4. **Git**  
   For branching: git worktrees. For PR creation: `gh` (GitHub) or `acli` (Bitbucket), depending on remote.

---

## File Layout

| Path | Purpose |
|------|---------|
| `~/.cursor/commands/swarm.md` | Slash command definition |
| `~/.cursor/agents/manager/config.json` | Workflow config (subagents, handoffs, behavior) |
| `~/.cursor/agents/planner.md`, `implementor.md`, etc. | Agent definitions |
| `~/.cursor/agents/*/`.schema.json | Result schemas |
| `.swarm/standards.md` | Project standards (in repo root) |
| `.swarm/runs/{run-id}/` | Run artifacts (plan, handoffs, results) |

---

## Troubleshooting

### "Standards are missing" or "Standards are empty"
- Create `.swarm/standards.md` at the project root.
- Include concrete rules: naming, exports, patterns, tooling (Bun vs npm, etc.).
- Remove placeholder text; Swarm rejects empty or generic content.

### "Working tree must be clean"
- Run `git status`; commit, stash, or discard changes.
- Untracked files are fine.

### Run stops at Planner with "plan rejected"
- Planner output didn’t pass plan validation (missing tasks, too broad, conflicts with standards).
- Review the feedback and adjust your task or standards.
- If replanning is exhausted (`maxReplans` reached), outcome is `needs-human`.

### Reviewer keeps rejecting
- Check `.swarm/runs/{run-id}/reviewer.result.json` for `issues`.
- Ensure `.swarm/standards.md` is clear and consistent with what Reviewer expects.
- After `maxLoops` (default 3) reject→fix cycles, Swarm escalates to `needs-human`.

### Implementor reports "partial" or "failed"
- Often due to lint/typecheck failures.
- Check `implementor.result.json` for `checks.lint`, `checks.typecheck`, and `blockedReason`.
- Fix issues manually or refine the handoff and re-run.

### Tester fails
- Check `tester.result.json` for `failures`, `output`, `exit-code`.
- Ensure `.swarm/standards.md` specifies the correct test command.
- If standards say "use Bun" but tests run with npm, Tester will fail.

### PR creation fails
- Swarm uses `gh` for GitHub, `acli` for Bitbucket (inferred from remote URL).
- Ensure the correct CLI is installed and authenticated.
- Run does not fail if PR creation fails; it records a warning. Create the PR manually from the pushed branch.

### Schema validation failed
- A subagent’s `.result.json` didn’t match its schema.
- Inspect the run folder for malformed JSON or missing required fields.
- Check `~/.cursor/agents/{agent}/` schemas and update agent behavior if needed.

### Worktree errors
- Ensure sibling directory is writable (worktrees are created at `../<workspace>-swarm-<name>`).
- If a previous run crashed, an orphan worktree may remain. Remove it with `git worktree remove <path>`.

### Run folder is cluttered
- Old runs live in `.swarm/runs/`. You can delete run folders for completed or failed runs.
- Add `.swarm/runs/*` to `.gitignore` if you don’t want run artifacts in version control.

---

## Config Reference

See `~/.cursor/commands/CONFIG.md` for detailed config field documentation.
