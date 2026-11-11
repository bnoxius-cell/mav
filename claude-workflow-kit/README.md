# Claude Code subagent workflow kit

A router setup: the main session picks the cheapest model that can do each job, a tester checks the result, and it only steps up to a stronger model when a check fails. Saves a lot of usage on a Pro plan.

## Install (2 minutes)
1. Copy the `agents` folder to `~/.claude/agents/` (Windows: `C:\Users\<you>\.claude\agents`).
2. Copy the `docs` folder to `~/.claude/docs/`.
3. Open `~/.claude/CLAUDE.md` (create it if missing) and paste in the contents of `CLAUDE-snippet.md`.
4. Start a new Claude Code session. Run `/agents` to confirm the seven agents show up.

## Using it
Just ask for things normally. Claude states a route like "Route: planner → builder-sonnet → tester", then runs it. Small fixes it does itself. Builders work on their own git branch and commit but never merge, so use it inside a git repo.

## Tweaks
- Set the main session model with `/model` (Sonnet, medium effort works well).
- Add project specifics (test commands, ports) to each project's own `CLAUDE.md`. The agents read it.
- Edit any agent file to change its model or rules.

Idea source: the "GOAT" router, instagram.com/p/DeOaTjWMOe2.
