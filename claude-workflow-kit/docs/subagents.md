# Routing, subagents and stepping up

**Every task and project runs this way.**

Idea source: the "GOAT" router from instagram.com/p/DeOaTjWMOe2 (Daniyar Kazi). Same six tasks, cost per solved task:
- Top model at max effort for everything: $5.10.
- Hand-picked model, always high effort: $0.75.
- Cheapest first, verify, escalate on failure: $0.45.
- A cheap router picks model and effort per task: $0.32.

The rules that come from it:
- A cheap step (the orchestrator) reads each task and picks the model and effort.
- A clear spec does the thinking up front, so well-specified work runs on cheap rungs at low effort.
- The top tier is for hard work and for when a check fails, never the default.

Standing setup. The main session is the **orchestrator** on Sonnet medium. It never switches models: planning goes to `planner` (always Opus), building to a builder picked by difficulty, checking to `tester`. Subagents never talk to each other; every report comes back to the orchestrator, which decides the next hop and passes it on.

## Who does what
| Who | Model | Does |
|---|---|---|
| Orchestrator (main session) | Sonnet medium | Routes every task, talks to the user, relays between subagents, does small fixes itself, verifies, merges, keeps the docs |
| `planner` | Opus high | Investigates and writes the plan (batches, acceptance checks, decisions). Read-only except the plan file |
| `builder-haiku` | Haiku medium | Purely mechanical batches with an exact brief |
| `builder-sonnet` | Sonnet medium (low for mechanical-but-fiddly) | Normal plan batches with clear acceptance |
| `builder-opus` | Opus medium (high for hard bugs) | Tricky logic, unfamiliar areas, concurrency, migrations, hard bugs, step-ups |
| `tester` | Picked per test: Haiku / Sonnet (default) / Opus | Runs the plan's acceptance checks plus edge cases, reports Pass/Fail with repro. Never fixes |
| `code-reviewer` | Opus high, read-only | Reviews a branch before merge. Verdict: Merge or Fix first |
| `Explore` | Haiku | Wide read-only searches. Verify its key claims before acting on them |

Definitions are global in `~/.claude/agents/`. Project specifics (check commands, ports, test tools, doc locations, routing notes) go in the project's CLAUDE.md. Don't add same-named agents in a project unless it truly needs a different one.

## The loop
1. **Route.** The user asks for something. Orchestrator states the route in one line ("Route: planner → builder-opus → tester, full verify"). Quick answers and one-file fixes it just does itself.
2. **Plan.** Anything bigger than a small fix goes to `planner` with the goal, symptoms, screenshots/paths and decisions already made. Planner returns a plan file plus open decisions. Orchestrator asks the user (AskUserQuestion), records answers in the plan, and SendMessages them to the planner if the plan needs revising.
3. **Build.** Orchestrator spawns the builder the plan suggests (sanity-check it against the table below), one batch per spawn, pointing at the plan section.
4. **Test.** Orchestrator spawns `tester` with the plan's acceptance checks for that batch.
5. **Fail → fix.** Tester reports to the orchestrator. Orchestrator decides:
   - Cause is clear and local (tester named the line, fix is obvious): straight to a builder fix round with the findings.
   - Cause unclear, or the plan was wrong: findings go to `planner` for a fix plan, then to a builder.
   - Each fix round goes one rung up (see Stepping up), then back to step 4.
6. **Pass.** Run `code-reviewer` if a trigger applies, merge , update docs and routing notes.

Two failed fix rounds on the same batch at `builder-opus` high: stop and bring it to the user with the tester's findings.

## Picking a builder
How tight the plan is decides the rung more than how big the task is. A precise plan lets hard-sounding work run cheap. Small fixes get no builder; the orchestrator does them.
- **`builder-haiku`** (Haiku medium): purely mechanical work from an exact brief, like renames, copy text, config values, or a numbered fix list with paths and exact changes. If a step needs judgment it stops and reports "Needs a decision", which means the work goes to `builder-sonnet` low.
- **`builder-sonnet`** (Sonnet medium, low for fiddly mechanical work): the default for normal plan batches and features with clear acceptance, like a UI panel, a feature across a few files, or a planner batch. It's also the first step up after a `builder-haiku` failure.
- **`builder-opus`** (Opus medium, high for hard bugs): tricky logic, unfamiliar areas, concurrency, migrations, hard bugs a brief fully describes, and anything that failed on `builder-sonnet`. Use it for core logic and tricky timing. On fix rounds it reproduces the tester's failure first.
- **Start cheap.** The planner's suggested builder is a ceiling, not the pick. When the plan has exact files, traps and acceptance tests, start one rung below the suggestion (suggested opus → `builder-sonnet` high; suggested sonnet → `builder-sonnet` low/medium). `builder-opus` first is for (a) code where a bug would be a security hole (auth, passwords, tokens, cookies, permissions, reading or serving other users' data, uploads, signed URLs; the same triggers as `code-reviewer`), (b) work with no usable spec, or (c) after a cheaper rung failed its test. Everything else starts on Sonnet or Haiku, concurrency and size included. State the rung and why in the route line.
- **All builders:** one brief, own branch (or a backup step for non-git projects), commit, never push or merge. They stop after two failed fix attempts and report, and the next round goes one rung up.
- **Builders build, testers test.** A builder runs only the cheap compile checks (the project's build and unit-test commands, lint if it touched lint-relevant files) and nothing else. No browser harness, no route sweeps, no contrast or keyboard passes, no screenshots, no detector runs, unless the brief asks for one specific look. Its report says what changed and what it did NOT verify. The `tester` does all verification, independently. Say this in every builder brief, and don't paste the plan's full check list into it; give the tester that list.

## Quality comes before cost
Everything below about cheap rungs and saving tokens serves one goal: clean code and a working system with few bugs. When they conflict, quality wins.
- Never accept a cheap rung's output just because it passed once: the orchestrator reads the diff, the tester runs the acceptance checks, and anything shared or security-sensitive also gets `code-reviewer`.
- If a cheap run leaves a mess (copied bad patterns, duplicated logic, thin tests, "done" claimed without running it), send it back or move up. Don't merge it to save a round.
- Don't skip the tester, the diff read or the review to save tokens. Save tokens by cheaper building and tighter briefs, never by weaker checking.
- Unsure whether a batch is safe on a cheap rung? Pick the stronger one. A bug found in production costs more than the extra tokens.

## Choosing the rung: decision order
Run these top to bottom for every batch and stop at the first hit. Say the result and the reason in the route line ("builder-sonnet high: spec is exact, no security surface").
1. **Security surface?** (auth, passwords, tokens, cookies, permissions, other users' data, uploads, signed URLs, money) → `builder-opus` high, then `code-reviewer`. This overrides everything below.
2. **Already failed a test on a lower rung?** → pick the next rung by diagnosing the failure, not by habit (see "Stepping up": one rung for slips, jump to Opus for real difficulty).
3. **Pure mechanical?** (exact old → new text, renames, config values, copy, a numbered list of edits with paths) → `builder-haiku`. Needs any judgment → `builder-sonnet` low.
4. **No usable spec?** (unfamiliar area, design decisions still open, root cause unknown) → `planner` first, not a builder. If a quick read settles it, `builder-opus` medium.
5. **Otherwise** (the plan names files, traps and acceptance tests) → `builder-sonnet`. Effort: low for repetitive edits, medium default, high for many files, tricky state or a long acceptance list.
Tie-breaker: if two rungs fit, pick the cheaper and tighten the brief. A failed test costs one step-up round, which is cheaper than starting high every time.
Signals that push UP: shared state or concurrency across requests, data migrations, a bug the brief can only describe by symptom, a change that must not break many callers. Signals that push DOWN: a plan with exact acceptance tests, a pattern already used nearby, a single layer (only server or only CSS).
The planner's suggested builder is a ceiling, never the pick. The same applies to the tester and reviewers: choose their model by the "Every spawn picks its model" list below, not by habit.
**Using Opus without wasting tokens**:
- Opus is a scalpel: give it only the hard part (the security-sensitive or tricky module) and send the plumbing, tests-for-plumbing and wording to Sonnet or Haiku in a separate batch when the plan can be split cleanly.
- Effort: medium by default, high only for security code or a bug the brief can't pin down. Never xhigh without the user.
- Keep the brief short and exact: plan section, files, traps, decisions. Point at files, don't paste them. No exploring the whole repo "to understand".
- One spawn per batch. If it stops or needs an answer, continue it with SendMessage instead of re-spawning (a cold start re-reads everything). If I stop an Opus run, check `git status` first so nothing is lost.
- Don't run an Opus builder and then an Opus reviewer on the same small diff: tester first, `code-reviewer` only on the security triggers or at the end of a branch.
Learn from outcomes: when a rung passes first time, note it in the project's routing notes so similar batches start there; when one fails, note why.

## Routing table
Check the project's routing notes first. If a task sits between rows, tighten the brief so it fits the cheaper one.

| Task | Route | Verify |
|---|---|---|
| Quick answer, git chores, docs, one-file fix | Orchestrator | light |
| Wide search ("where is X used") | `Explore` | check key claims |
| Plan, architecture, schema/API design, root-causing a bug | `planner` | user approves |
| Mechanical batch with an exact brief | `builder-haiku` (or `builder-sonnet` low if it needs any judgment) | light |
| Normal plan batch with clear acceptance | `builder-sonnet` | tester |
| Tricky logic, unfamiliar area, concurrency, migrations | `builder-opus` medium | tester + read the diff |
| Hard bug a brief can fully describe | `builder-opus` high | tester + read the diff |
| Security-sensitive diff before merge | `code-reviewer` | is the full verify |

**Every spawn picks its model.** On each spawn, choose Haiku, Sonnet or Opus (and effort) for that specific task and say it in the route line. Planner is the only fixed one (always Opus). For builders, pick the agent that matches (`builder-haiku` / `-sonnet` / `-opus`). For everyone else, pass `model`:
- `tester`:
  - Haiku: running a test command or scripted check and comparing the output.
  - Sonnet (default): UI checks that need judgment.
  - Opus: hard-to-reproduce bugs, timing or concurrency, or a fix round that already failed once.
- `Explore`:
  - Haiku (default): finding where something is.
  - Sonnet: when it has to understand how code fits together.
- `code-reviewer`:
  - Opus high (default) for security triggers.
  - Sonnet for a non-security review the user asks for.

Spawn builders only when the work would bloat the orchestrator's context; small and medium fixes stay in the main session. One builder at a time; parallel builders need the user's OK. Pass `effort` on a spawn only when it differs from the agent's default.

## Verify by risk
- **Light** (copy, styling, config, isolated changes): checks pass, report reads right, `git diff --stat` matches the brief.
- **Tester** (anything users touch, shared logic): `tester` runs the acceptance checks. Add a diff read for `builder-opus` work and run `code-reviewer` if a trigger applies.
- When unsure, use the tester.

## Stepping up
Ladder: `builder-haiku` → `builder-sonnet` low → `builder-sonnet` medium → `builder-opus` medium → `builder-opus` high → `builder-opus` xhigh. Beyond that only with the user's say-so. The ladder is an order of cost, not a path you must walk: jump rungs when the failure says so (below).
- **Diagnose before stepping.** Read the tester's findings and sort the failure, then pick the rung. Walking every rung (haiku → sonnet → opus) costs three runs; jumping straight to the right one costs one.
  - **Slip** (typo, missed file, one wrong value, a case the brief didn't list, a lint or build error): same rung or one up with the exact fix list. Cheap.
  - **Real difficulty** (wrong logic or model of the problem, state or ordering bug, a fix that broke something else, the builder says "needs a decision", or two slips in a row on the same area): skip straight to `builder-opus`. A cheaper rung will most likely fail the same way and burn tokens twice.
  - **Security finding** (tester or reviewer): `builder-opus` high, no matter which rung built it.
  - **Brief or plan was wrong or unclear:** fix the brief (via `planner` if the design is off) and rerun on the same rung. Don't pay for a stronger model to guess around a bad spec.
- Haiku that stops with "Needs a decision" is not a failed test: go to `builder-sonnet` low with the decision written in. A Haiku test failure that is a slip goes to Sonnet; one that is real difficulty goes to Opus.
- At the start, use the expected cost: if a batch has more than about a 40% chance of failing on the cheap rung (hard logic, shared state, many callers), the cheap attempt plus the Opus retry costs more than Opus once. Start on Opus medium for those, scoped to just the hard part.
- Opus keeps clean code as the goal, so use it where the code is hard to get right, not as the default for everything. Plumbing it writes costs the same tokens as plumbing Sonnet writes, at a higher price.
- Same for the orchestrator: if a fix fails twice in the main session, brief it to a builder one rung above where it would normally route.

## Context budget (when to /clear)
Subagents have their own context; only their reports land in the orchestrator's. So the orchestrator's window grows slowly, and the user can `/clear` less often. Every turn re-sends the whole window, so a big one costs usage and makes the orchestrator vaguer.
- The orchestrator estimates its own context size (it has no exact counter; judge by how much has been read and relayed, or the status line if the user shows it).
- **About 100k:** at the end of the reply, after the answer, add one line: "Context is getting big (~100k). Good point to /clear after the next merge or state-file update."
- **About 150k:** say it first and plainly: "Context ~150k. Please /clear now."
- Before either warning, make sure the project's state file is current, so nothing is lost. Say it's updated.
- Don't warn mid-batch while a subagent is running; wait until its report is relayed and the state is saved.
- The user can set different numbers; follow them.

## Learning
When a task had to step up, or a cheaper rung clearly would have done, add one line to "Routing notes" in the project's CLAUDE.md (e.g. "payment webhook logic: start at builder-opus medium"). Keep it to about 5 lines; replace stale ones.

## `code-reviewer` triggers
Anything touching auth, passwords, reset codes, tokens, cookies, encryption, signed URLs, permissions, other users' data, money/purchases; anything the project's CLAUDE.md flags; and whenever the user asks.

## Writing a brief (orchestrator → any subagent)
It has to stand alone; the tighter it is, the lower the rung it can run on:
- What to do: plan file and section, or a numbered list with file paths and expected behavior.
- Branch name or "continue branch X" (or a backup step for non-git projects).
- Decisions from chat the plan doesn't record yet (better: record them first, then point at the plan).
- For fix rounds: the tester's findings verbatim (steps, expected vs actual, logs), and the rung.
- Every place a fix applies. A builder fixes exactly the files named.
- What not to touch, known traps, tool setup.
- Code quality (priority #1): "Follow `~/.claude/docs/code-quality.md`. Write clean, well-structured code; learn the project's conventions from nearby files but don't copy bad patterns." Ask the report to list files changed, what was reused, what was verified and not, and any smell noticed but left alone. The orchestrator reads the diff before accepting.
Point at files instead of pasting them.

## Running and merging
1. Commit your own changes first: the tree must be clean when a builder starts. Don't edit the tree while it runs.
2. Build → test loop above.
3. Push the branch, merge into main, push main.
4. Update the project docs from the reports (state file, plan checkboxes, to-do list) and routing notes if anything stepped up.
