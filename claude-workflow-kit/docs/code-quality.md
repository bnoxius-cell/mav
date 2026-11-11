# Code quality (priority #1, alongside the agent workflow)

The user cares how Claude writes code. Clean, working code beats fast code. This applies to the orchestrator and to every subagent, and it goes into every builder brief.

## Rules
- **Write good code, whatever is around it.** Clean, simple, well-structured, no spaghetti. Quality is the standard, not the surrounding code. Read the neighbouring files to learn the project's conventions (naming, folders, tokens, shared helpers) so new code fits and works, but never copy a mess. If the nearby code is ugly, write the new code well anyway, keep it consistent only where that doesn't mean copying a bad pattern, and note the ugly part in the report instead of rewriting it unasked.
- **Reuse before adding.** Use the existing component, helper, token or pattern. No new library, abstraction or file unless the existing ones can't do the job.
- **Smallest change that solves it.** Touch only what the task needs. No drive-by refactors, no speculative options, no dead code, no commented-out code, no leftover debug logging.
- **Correct first.** Handle the real edge cases (empty, missing, bad id, other user's data, slow or failed network). Validate on the server. Never swallow an error: log it and answer with a plain message.
- **Readable.** Clear names over comments. Short functions that do one thing. Comments only for why, never for what. No clever tricks.
- **No duplication, no sprawl.** If the same logic appears twice, share it. Keep files focused; split a file that grows several jobs.
- **Prove it works.** Builders run the project's build and unit tests and fix every failure before committing. The `tester` verifies behaviour independently. A report says what was verified and what was not. Never claim done on code that wasn't run.
- **Tests with the code.** New server logic ships with tests (happy path, bad input, usership). Fix flaky or failing tests; never delete or loosen one to get green.
- **Safe by default.** Nothing trusted from the browser, secrets never in client code or logs, least privilege, usership checks on every resource.
- **Leave it better and tell the truth.** If you find a bug or smell you weren't asked to fix, note it in the report; don't silently expand scope.

## How it's enforced
- Every builder brief includes: "Follow `~/.claude/docs/code-quality.md`. Write clean, well-structured code; learn the project's conventions from nearby files but don't copy bad patterns."
- Every builder report lists: files changed, what was reused, what was verified, what was not, and any smell noticed but left alone.
- The orchestrator reads the diff (`git diff --stat`, then the real diff for anything non-trivial) before accepting a batch, not just the report.
- `code-reviewer` runs for security triggers (see `subagents.md`) and also when a diff looks bigger or messier than the brief justified.
- A fix round is for the tester's findings only. Quality problems the orchestrator spots in the diff go back to the builder as a numbered list.
