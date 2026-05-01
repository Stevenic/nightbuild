# NightBuild

[![Agent Ready](https://img.shields.io/badge/Agent-Ready-blue.svg)](#agent-ready)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Hand off a coding task at bedtime. Wake up to a working artifact and a clean post-mortem.

---

## Overview

NightBuild lets you hand a coding task to an AI agent at bedtime and wake up to a working artifact, a clean post-mortem, and concrete commands to commit or open a PR. Queue what you want done in `TONIGHT.md` across the day, paste a one-line prompt into your coding agent before bed, and find a `MORNING_TODO.md` waiting for you in the morning — wake-up summary plus copy-paste `git`/`gh` commands to keep, merge, or discard the night's work.

Under the hood the agent generates a per-run program (mission, phases, done-bars) from your queued tasks, then runs an autonomous loop that works phase by phase — each phase gated by a testable done-bar, each tick committed, each iteration reflected on so the loop adapts to its own mistakes. The full per-iteration narrative is preserved under `.nightbuild/<date>/` for post-mortem.

**Best fit:** tasks with a clear acceptance criterion ("the smoke test exits 0", "the installer produces a working `.exe`", "all tests green") that decompose into 4–8 phases. **Worst fit:** research-heavy work or ambiguous specs that need human judgement at every step.

**Optional knobs:** branch isolation for trivial morning rollback, Slack / OS-notification / email when the run finishes, a dollar-cost cap, and a project-scoped `learnings.md` corpus that compounds knowledge across nights so future runs anticipate this project's gotchas (flaky tests, slow steps, surprises).

---

## Agent Ready

NightBuild ships a canonical [`NIGHTBUILD.md`](NIGHTBUILD.md) — the operational spec a coding agent reads to set up and run a NightBuild on your project. Paste this prompt into your agent of choice:

```
Read the NIGHTBUILD.md file at https://raw.githubusercontent.com/Stevenic/nightbuild/main/NIGHTBUILD.md
and follow its instructions to kick off a night build for this project.
```

The agent picks the right flow based on the project state:

- **First time on this repo** *(no `TONIGHT.md` yet)* → it runs **Setup**: asks a couple of questions (mostly about `.gitignore`), creates `.nightbuild/`, writes the initial `TONIGHT.md`, and shows you a quick overview of how to use NightBuild from here on. Setup ends without starting a run.
- **Subsequent invocations** *(TONIGHT.md exists)* → it runs **Kickoff**: drains `TONIGHT.md`, asks clarifying questions, generates a per-run program at `<run_dir>/program.md`, runs an in-conversation **boot tick** so prereq failures surface before you sleep, schedules the autonomous loop, and hands you a `MORNING_TODO.md` in the morning.

That single file is self-contained. It covers the setup flow, the kickoff flow, the in-conversation boot tick, the per-tick recipe (with crash recovery + version-pinned recovery on context loss), reflection structure, wakeup pacing, stop conditions (wall-clock budget + optional cost cap), authority/boundary defaults, the per-run program schema, and embedded scaffolds for `TONIGHT.md`, `state.json`, `preferences.json`, and the project's append-only `learnings.md`. The agent reads it once and works from conversation context after that; it never writes a copy of `NIGHTBUILD.md` into your project.

**Optional preferences** *(set by editing `.nightbuild/preferences.json`)* — fire a Slack/OS-notification/email when the run finishes via a `notify` shell command, set a `usd_per_mtok` rate to track estimated dollar cost across ticks, and pair it with a kickoff-time `$10`/`$25` cap to stop automatically if the loop overspends. None of these are required.

---

## Premise

NightBuild is a file-driven autonomous-loop pattern for software work that:

- has a clear, declarable acceptance criterion ("a working installer", "a deployed service", "all green tests");
- can be decomposed into a small ordered sequence of phases each with its own done-bar;
- benefits from reflection-driven self-improvement *within* a single overnight run.

A NightBuild run has two phases: a short **kickoff** in-conversation (the agent reads the canonical [`NIGHTBUILD.md`](NIGHTBUILD.md), drains `TONIGHT.md`, asks clarifying questions, and writes a per-run **program**), then an **autonomous loop** that runs while the user sleeps. Each tick the loop reads the program at `<run_dir>/program.md` and its state at `<run_dir>/state.json`, executes the next sub-step of the current phase, verifies, commits, *reflects* into the run-scoped log under `.nightbuild/<YYYY-MM-DD>/`, updates state, and yields back to the scheduler. NIGHTBUILD.md stays in conversation context after kickoff — no per-tick re-fetch. Repeat until success or stop conditions trip.

**Three properties make this work:**

1. **Single source of truth.** The playbook is the contract. The loop never invents work outside it; the user never has to babysit.
2. **State + log files robust to context loss.** Every artifact persists to disk. A fresh agent invocation can resume any time by reading the state file.
3. **Reflection in every tick.** Each iteration ends with five short lines (what I did / what worked / what didn't / improvement for next pass / confidence). The next tick reads the recent reflections before planning, so the loop adapts to its own mistakes.

---

## File layout

`NIGHTBUILD.md` is **not** in the user's project. It lives in this repo and is read remotely on every tick. The user's project gets only the files below:

At the repo root:

| File | Purpose | Gitignore default |
|---|---|---|
| `TONIGHT.md` | Human-authored task queue. The user accumulates intent into it across the day; the agent drains it at kickoff (synthesizing the items into `<run_dir>/program.md`), then resets it to the empty scaffold so it's ready for tomorrow night's queue. | Tracked on solo repos; ignored on multi-person repos. |
| `MORNING_TODO.md` | Written when the success criterion is met (or the run terminates). The agent's morning summary — what landed, what's blocked, the concrete `git` commands to commit/PR/discard the night's work, and resume instructions. | Tracked. |
| `STOP` *(optional sentinel)* | If the user creates this file, the loop stops cooperatively next tick. | Ignored. |

Under the top-level `.nightbuild/` directory (always gitignored — these are local-only artifacts that don't belong in version control), two cross-run files plus per-run dated folders:

| Path | Scope | Purpose |
|---|---|---|
| `.nightbuild/learnings.md` | cross-run | Append-only project corpus — one block per run, recording load-bearing lessons (flaky tests, slow steps, surprises). Read at kickoff so the agent anticipates this project's gotchas; appended-to in the End-of-overnight protocol. Never auto-pruned; user can edit. |
| `.nightbuild/preferences.json` *(optional)* | cross-run | Per-clone preferences: `branch_policy`, `notify` shell command (Slack/OS-notification/email on terminal events), `usd_per_mtok` cost rate. Created lazily on the first "always"/"never" answer or whenever the user manually configures notifications/cost. |
| `.nightbuild/<YYYY-MM-DD>/` | per-run | The `run_dir` for tonight's run (suffix `-HHMM` if multiple runs in one day). Each run is fully self-contained inside its own folder. |
| `.nightbuild/<YYYY-MM-DD>/state.json` | per-run | Machine-readable state. Seeded at kickoff, read at tick start, written at tick end. Includes the pinned NIGHTBUILD.md SHA for context-loss recovery, a `tick_in_progress` flag for crash recovery, and cumulative `tokens_total` / `usd_estimate`. |
| `.nightbuild/<YYYY-MM-DD>/program.md` | per-run | The per-run program — the plan the loop executes. Generated at kickoff from `TONIGHT.md` + clarifying-question answers. Mission, phase plan, scope deviations, authority overrides, budget. Read every tick. Schema is in [`NIGHTBUILD.md`](NIGHTBUILD.md) § "Program format". |
| `.nightbuild/<YYYY-MM-DD>/log.md` | per-run | Append-only post-mortem-shaped log. Reflections + decisions + iteration history for this run. |
| `.nightbuild/<YYYY-MM-DD>/handoff.md` | per-run | Pre-composed commit subject + PR-body markdown. First line is the commit subject and PR title; lines below are the body. Written at the end of the run; pipe into `git commit -F` or `gh pr create --body-file`. |
| `.nightbuild/<YYYY-MM-DD>/raw/` | per-run | Raw command output, build logs, smoke-test artifacts captured during the run. |

### Per-run program structure

Each run's program lives at `<run_dir>/program.md` and is generated by the agent at kickoff. The loop executes it tick by tick. Sections (full schema in [`NIGHTBUILD.md`](NIGHTBUILD.md) § "Program format"):

- **Mission.** What "done" looks like — concrete artifact path + observable assertion.
- **Scope deviations.** What's being substituted from the spec and why; architectural invariants retained.
- **Inputs.** Captured Q&A from kickoff. The loop refuses to start with required inputs missing.
- **Phase plan.** Ordered phases A, B, C... with testable done-bars. Each done-bar runs without human judgement.
- **Authority overrides.** Anything that diverges from NIGHTBUILD.md's allow/deny defaults for this project.
- **Budget.** Wall-clock cap (default 6 hours) + start timestamp.

---

## The tick recipe

The full tick recipe is specified in [`NIGHTBUILD.md`](NIGHTBUILD.md) § "Tick recipe" — that file is canonical for behavior. The summary below is the human-readable version.


```
1. Read <run_dir>/state.json. (state.run_dir was set at kickoff to .nightbuild/<YYYY-MM-DD>/.)
2. Check stop conditions (§ 6). If tripped, write final entry, exit without scheduling.
3. Pick next sub-step within the current phase. Keep sub-steps under ~20 min of wall-clock work.
4. Read recent reflections (last 3) from <run_dir>/log.md. If a recurring blocker pattern is
   visible, incorporate the lesson into this step's plan.
5. Do the work. No exploration of unrelated parts of the repo.
6. Verify: build, unit test, smoke test for the step. Stream raw output to
   <run_dir>/raw/<phase>-<iteration>.log so context stays small. If failed, iterate up to 3
   times within this tick. After 3 failures, write a BLOCKED entry and yield without
   advancing state. Increment consecutive_blocked.
7. Commit. `git add -A && git commit -m "<phase>: <one-line summary>"`. Never bypass
   hooks. Never --amend. Never push.
8. Reflect (see § Reflection structure below). Append to <run_dir>/log.md.
9. Update state: increment iteration, set last_completed_step, append history entry,
   reset or increment consecutive_blocked.
10. Append iteration log section to <run_dir>/log.md (separate from the reflection — covers
    commands run, state delta, decisions logged, next step).
11. Schedule next wakeup with appropriate delaySeconds (§ Wakeup pacing).
```

## Reflection structure (every tick)

```markdown
### Reflection — iteration N

**What I did:** one sentence.
**What worked:** one bullet — concrete win to repeat.
**What didn't:** one bullet — concrete miss to avoid; null if nothing notable.
**Improvement for next pass:** one bullet — actionable change to the next tick's approach.
**Confidence in current phase done-bar:** low / medium / high, with one-line justification.
```

Why short: long reflections waste context. The "improvement for next pass" bullet is the part future ticks read in step 4 — keep it actionable.

Why "what worked" not just "what didn't": pure-correction reflections push the loop toward overcaution. Recording successes preserves validated approaches.

## Wakeup pacing

The Anthropic prompt cache has a 5-minute TTL. Sleeping past 300 seconds means the next wakeup reads the conversation context uncached.

- **Active phases (compile errors, fast feedback):** `delaySeconds: 60`–`120`. Cache stays warm.
- **Longer-running steps (installer build, full smoke):** `delaySeconds: 120`–`270`. Still in cache.
- **BLOCKED + need diagnostic time:** `delaySeconds: 270`.
- **Idle waiting on something specific:** `1200`–`1800` (one cache miss buys a longer wait).

Never `300` (worst-of-both). Never past `1800` for an active project.

## Stop conditions

Stop and do not reschedule when ANY of these is true:

- All phase done-bars are green AND the morning-readiness gate (final smoke) passes.
- `consecutive_blocked >= 3`. Write `STOPPED: stuck on <step>`. Create a top-of-log `## NEEDS HUMAN` block describing the blocker.
- Wall clock since iteration 1 exceeds your budget cap (recommend 6 hours).
- A required input is missing from state.
- User creates a `STOP` sentinel file.
- Branch has uncommitted changes the loop didn't make (defensive — pause rather than overwrite).

On success, write a top-of-log `## MORNING SUMMARY` to `<run_dir>/log.md`. On block, write `## NEEDS HUMAN`. Do this *immediately* on hitting the criterion, not at the end of the run.

## Authority and boundaries

Default allow-list:

- Edit files inside the project.
- Run language/framework tooling locally (`npm`, `tsc`, language-specific build).
- Run unit + smoke tests locally.
- Install language packages from public registries.
- `git init`, `git add`, `git commit` (in the project).

Default deny-list (require explicit user opt-in):

- Push to any remote (`git push`, `gh pr create`, equivalent).
- Force-push.
- Delete files outside the project.
- Skip git hooks (`--no-verify`).
- Disable signing, disable security baselines.
- Modify global git config or shell profile.
- Touch external-account state (Stripe products, OAuth client IDs, cloud resources).

If the loop encounters something it would normally need to bypass (a failing pre-commit hook, a missing dependency from outside the project), it must STOP and surface the blocker. Do not "make it work" by disabling safeguards.

## Logging hygiene

- Never log secrets, API keys, tokens, or paths inside identity stores.
- Truncate test/build output longer than 20 lines: keep first/last 5 with `... <N lines elided> ...` between.
- Commit messages: `phase: short summary`. No emojis.
- Append-only on logs — do not edit prior entries (the post-mortem narrative is built from them).

---

## Patterns that worked in the prototype run

These are the load-bearing techniques the Stillframe Desktop run validated. Use as defaults when adapting NightBuild to a new project:

### Structured-501 endpoints for service scaffolds

Scaffold service routes with full request validation but a placeholder body:

```ts
{ error: 'unimplemented', message: 'X not yet implemented — set ENV_VAR_FOO,ENV_VAR_BAR to enable.', unimplemented: true }
```

Validation (400) and auth (401) fire BEFORE the 501 stub. This lets the test suite lock the wire shape today; future implementation work is purely the body, not the surface. The contract doc + tests serve as a self-documenting TODO that pairs cleanly with the NightBuild loop's reflection model.

### "Try once, observe error, fix, retry" beats "design for all cases"

The run's most efficient iterations were ones where I wrote the smallest plausible thing, ran it, read the error message, and fixed exactly that. Every attempt to "anticipate all the failure modes" before running cost more than it saved. Trust the error message; don't pre-architect around imagined ones.

### Delegate API discovery to a subagent

Reading 13K lines of source is expensive in main-context tokens. A read-only Explore subagent with a tight, specific prompt ("report the exact public API of X with file:line refs") produces a 700-word reference doc that consumes 1/5 the tokens of inline reading.

Pattern:

```
Agent({ subagent_type: "Explore", prompt: "Read X. Report: 1. function signatures with file:line. 2. ... Under N words. Specific facts only." })
```

### Test-driven for interface compliance, not internals

When implementing against a third-party interface (a `StorageProvider`, a route protocol, an event schema), write tests that exercise the *interface surface* before writing the implementation. The tests are short and fast, and they catch shape-of-data mistakes that runtime smokes find too late.

### Run the slow smoke only when the engine path changes

The full LLM-round-trip smoke test was the morning-readiness gate but cost 30s per run. After phases that didn't touch the engine path (UI, settings, polish), the cheap boot-only smoke was enough. Decision rule: ask "does this change affect the path the slow smoke covers?" — if not, skip it.

### Write MORNING_TODO at success, not at end-of-run

Once the success criterion is met, write the wake-up artifacts *immediately* — before any stretch work. A run that hits success then runs out of budget on stretch without writing the morning hand-off is strictly worse than one that exits cleanly at success.

### Single tsconfig, one source-tree mirror

For projects with multiple compilation contexts (Electron main / preload / renderer), one tsconfig with `rootDir: src, outDir: dist` and the source tree mirrored under dist is simpler than per-context tsconfigs with their own rootDirs. Per-context configs cause `rootDir` conflicts the moment shared code crosses contexts.

### Inline runtime constants when sandbox / module boundaries forbid imports

Sandboxed Electron preload can't relative-require. Vendor-style isolation boundaries (cross-process, cross-iframe, cross-language) often have similar restrictions. Inline the small set of runtime constants on both sides + a unit test that asserts they match. Cheap, robust, and the test is the spec.

### Pre-commit checks as morning-deliverable proofs

The packaged-binary smoke test proved Phase F at commit time, not at end-of-run. The morning-readiness gate became "the test that just passed" rather than "the test we need to remember to run later". This made the success criterion auditable from any commit.

### Pack vendor deps as tarballs, not symlinks

`file:../some-sibling-repo` works for development but breaks packagers (electron-builder, etc.) that follow the symlink and find files outside the project. Pack the dep as a tarball (`npm pack --pack-destination`), check the tarball into `vendor/`, install with `file:./vendor/X.tgz`. The tarball respects the dep's own `files` whitelist so the install is small and the package boundary is clean.

---

## Patterns that didn't work / lessons

### Don't pre-architect around imagined failure modes

I caught myself trying to add lockfiles, quotas, and backup logic to the storage provider all at once. Half of that was theoretical (single-process desktop). The single-feature increment ("just add backup-on-write") was the right scope; the rest was deferred to v1 with explicit rationale. **Lesson:** when a phase plan says "implement X with features A, B, C", ask which of A, B, C are theoretical vs load-bearing for the current scope.

### Don't trust spec language without verification

The spec said "save to `page.html`". The code actually saves first-edit pages to `page.cN.html` (it's a journal, not an overwrite). Read the actual save path in source before writing the assertion. **Lesson:** spec docs describe intent; code is authoritative.

### Don't do "all-or-nothing" work in the same tick

Phase E (packaging) had three independent fixes: symlink-out-of-tree, 7z+symlinks, app-name resolution. I tried to fix all three before testing. Testing earlier would have caught the second and third issues with cleaner error messages. **Lesson:** narrow the change, run, observe, narrow again.

---

## Setup and kickoff flows

The agent runs one of two flows depending on whether NightBuild has been bootstrapped on this repo. Canonical step-by-step is in [`NIGHTBUILD.md`](NIGHTBUILD.md) §§ "Setup flow" and "Kickoff flow"; this is the user-facing summary.

### Setup *(first time on a repo)*

What the user does:

1. **Paste the [Agent Ready](#agent-ready) prompt** into your coding agent. The agent detects there's no `TONIGHT.md` and runs setup.
2. **Answer two questions** — confirm the repo, and (on multi-person repos) whether to gitignore `TONIGHT.md`.
3. **Read the overview** the agent prints when it's done.

What the agent does:

- Confirms the repo is a git repo (offers `git init` if not).
- Detects solo vs multi-person repo (`git log` authors, presence of checked-in `.gitignore`).
- Configures `.gitignore` — always adds `.nightbuild/`; asks about `TONIGHT.md` on multi-person repos.
- Creates `.nightbuild/` and writes the initial `TONIGHT.md` from the embedded scaffold.
- Prints a "here's how to use NightBuild" overview and exits without starting a run.

### Kickoff *(each night)*

What the user does:

1. **Across the day:** open `TONIGHT.md` and add tasks as ideas come up.
2. **Before bed:** paste the same [Agent Ready](#agent-ready) prompt. The agent detects `TONIGHT.md` exists and runs kickoff.
3. **Answer the agent's clarifying questions.** Stop when the agent says it has enough.
4. **Sleep.**

What the agent does (it reads NIGHTBUILD.md once at kickoff and works from conversation context after that — the file is self-contained, with all scaffolds embedded):

1. Reads `TONIGHT.md` and asks clarifying questions until each queued task has a concrete acceptance criterion. (If `TONIGHT.md` is just the empty scaffold, it elicits one from scratch.)
2. Reads `<repo>/.nightbuild/learnings.md` if present — past-run lessons that shape the clarifying questions and the program.
3. Pins the canonical NIGHTBUILD.md version by resolving `main` to a commit SHA (`git ls-remote`) and recording it in state, so context-loss recovery later in the night fetches the *exact* spec the run started under, never a newer one.
4. Asks once for an optional cost cap (e.g. `$10`).
5. Asks whether to put the run on a fresh `nightbuild/<YYYY-MM-DD>` branch (skipped if you've already saved a preference). Picking "always" / "never" writes `.nightbuild/preferences.json` so future runs don't re-ask.
6. Provisions `.nightbuild/<YYYY-MM-DD>/` and initializes `<run_dir>/log.md`.
7. Writes `<run_dir>/program.md` — the per-run program the loop will execute.
8. Seeds `<run_dir>/state.json` from the JSON embedded in NIGHTBUILD.md.
9. Drains `TONIGHT.md` — archives the pre-drain content into `<run_dir>/log.md` under "Kickoff inputs", then overwrites `TONIGHT.md` with the empty scaffold so the queue is ready for tomorrow night.
10. **Runs the boot tick in-conversation** — verifies tools are installed, the build currently passes, git is healthy, and the program parses. Fails fast and surfaces to you before scheduling anything, so a missing prereq doesn't waste the night.
11. Schedules the first wakeup with `<<autonomous-loop-dynamic>>` as the prompt sentinel.

### What the agent does in the morning

When the success criterion is met (or the run terminates on a stop condition), the loop writes:

- **`<run_dir>/handoff.md`** — pre-composed text whose first line is the suggested commit subject and PR title, with a body covering what changed, what didn't land, and how to validate. Pipe directly into `git commit -F` or `gh pr create --body-file`.
- **`MORNING_TODO.md`** at the repo root — the wake-up summary. Status, what landed, what's blocked, and (when in a git repo) a **Commit & PR** section with concrete `git`/`gh` commands. For runs on a `nightbuild/<YYYY-MM-DD>` isolation branch you get three options as runnable commands: keep + merge, keep + open a PR (using `handoff.md` as the body), or discard. For current-branch runs you get push or PR-open commands.
- A top-of-log block in `<run_dir>/log.md` — `## MORNING SUMMARY` (success) or `## NEEDS HUMAN` (blocked).
- A new entry appended to **`<repo>/.nightbuild/learnings.md`** — date, status, phases completed, wall clock, tokens/cost, and the load-bearing reflections from the night. Read at the next kickoff so the agent anticipates this project's gotchas. Compounding value across runs.
- The optional **notify hook** (if `.nightbuild/preferences.json` has a `notify` shell command) fires with `NIGHTBUILD_STATUS` / `NIGHTBUILD_SUMMARY` / `NIGHTBUILD_RUN_DIR` / `NIGHTBUILD_REPO` env vars set — Slack ping, OS notification, email, whatever you wired up. Stops you from sleeping through a 1am block.

The full per-iteration narrative is preserved in `<run_dir>/log.md` for post-mortem.

---

## Open questions for NightBuild v0.2

- Should reflections be queryable across runs? A meta-corpus of "what worked / what didn't" across many NightBuild projects could improve the next pattern revision. **Tentative shape:** a `~/.claude/nightbuild/reflections.jsonl` that the loop appends to with `{ project, phase, iteration, what_worked, what_didnt, improvement }`. The next NightBuild run reads the most recent N entries before each phase.

- How to handle work that genuinely needs an interactive decision mid-run? Today the loop must STOP and surface to the user. A "low-stakes interactive prompt" mechanism that uses `AskUserQuestion` in a non-blocking way could expand the set of projects NightBuild can complete unattended.

- Cross-project parallelism. NightBuild today is single-loop. A "fleet" mode where multiple projects run in parallel with shared learning could amortize the overhead of writing a playbook.

- A `/nightbuild` slash-command that wraps the kickoff prompt generically (drops the user straight into the canonical NIGHTBUILD.md flow without copy-pasting the Agent Ready prompt).
