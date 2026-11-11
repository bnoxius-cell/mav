# Add this to ~/.claude/CLAUDE.md (your global instructions)

## How every task runs
Every task runs through the orchestrator workflow in `~/.claude/docs/subagents.md`. Read that doc at the start of every session. The main session is the router: it states the route in one line, sends planning to `planner` (Opus), building to the cheapest builder that fits, and checking to `tester`. It relays every report itself and steps up one rung only when a test fails.

All code follows `~/.claude/docs/code-quality.md`: clean, reuse before adding, smallest change, tests with the code, never claim done on code that wasn't run.

Subagents live in `~/.claude/agents/` and are spawned without asking: `planner`, `builder-haiku`, `builder-sonnet`, `builder-opus`, `tester`, `code-reviewer` (security-sensitive merges or on request), `Explore` (wide searches). Quick answers and small fixes stay in the main session.
