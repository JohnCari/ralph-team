# ralph-team

A custom slash command for [Claude Code](https://claude.com/claude-code) that creates **Agent Teams** for collaborative task execution — designed to work inside the [ralph-loop](https://github.com/ghuntley/how-to-ralph-wiggum) plugin.

Instead of a single Claude session handling everything (and burning through context), `/ralph-team` spawns a coordinated team of agents, each with their own context window, working in parallel on different parts of your task.

## How It Works

When you invoke `/ralph-team "Your task description"`, Claude acts as a **team lead** and:

1. **Analyzes the task** to determine how many teammates are needed (2-5)
2. **Partitions files** so no two teammates touch the same file
3. **Creates a task list** with dependencies using TaskCreate
4. **Spawns an Agent Team** with detailed per-teammate prompts (Architect, Implementer, Tester, Reviewer archetypes)
5. **Coordinates in delegate mode** — monitoring progress, handling stuck teammates, verifying output
6. **Handles ralph-loop continuations** — detects iteration 2+ and respawns departed teammates
7. **Outputs the completion promise** only when all tasks are verified complete
8. **Falls back** to single-agent mode with task tracking if Agent Teams aren't enabled

## Installation

1. Copy `ralph-team.md` into your project's custom commands directory:

   ```bash
   # Project-level (available in one project)
   cp ralph-team.md /path/to/your/project/.claude/commands/ralph-team.md

   # Or global (available in all projects)
   mkdir -p ~/.claude/commands
   cp ralph-team.md ~/.claude/commands/ralph-team.md
   ```

2. Enable Agent Teams in your Claude Code settings (`~/.claude/settings.json` or project-level `.claude/settings.json`):

   ```json
   {
     "env": {
       "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
     }
   }
   ```

## Usage

### Standalone

```
/ralph-team "Build a REST API with authentication and tests"
```

### With ralph-loop (recommended)

```
/ralph-loop "/ralph-team Build a REST API with auth and tests" --max-iterations 10 --completion-promise "DONE"
```

This combines the ralph-loop's iterative persistence with agent teams' parallel collaboration — each iteration spawns a fresh team that picks up where the last one left off via the shared task list and git history.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI
- [ralph-loop plugin](https://github.com/ghuntley/how-to-ralph-wiggum) (for iterative loop usage)
- Agent Teams feature enabled (see Installation step 2)

## Team Archetypes

The team lead dynamically selects teammates based on task complexity:

| Role | When Used | Responsibilities |
|------|-----------|-----------------|
| **Architect** | Complex tasks needing design first | Explores codebase, produces design plan, defines file boundaries |
| **Implementer** | Always (1-3 based on work streams) | Writes code for exclusively assigned files |
| **Tester** | Tasks involving code changes | Writes and runs tests for implemented code |
| **Reviewer** | Multi-module or critical code tasks | Reviews completed work, verifies integration |

**Sizing guide:**
- Simple (single module): 2 teammates
- Medium (2-3 modules): 3 teammates
- Complex (cross-layer): 4-5 teammates
- Trivial (< 3 files): Falls back to single-agent mode

## How It Integrates with Ralph-Loop

The ralph-loop feeds the same prompt to Claude each iteration. `/ralph-team` handles this by:

- **Iteration 1**: Full team creation — analyze task, create task list, spawn teammates, begin work
- **Iteration 2+**: Continuation — check TaskList for progress, respawn departed teammates, resume uncompleted tasks, create additional tasks if gaps are found
- **Final iteration**: Verify all tasks complete, output the completion promise to exit the loop

## Learn More

- [Agent Teams documentation](https://code.claude.com/docs/en/agent-teams)
- [Ralph Wiggum technique](https://github.com/ghuntley/how-to-ralph-wiggum)
- [Claude Code](https://claude.com/claude-code)

## License

MIT
