# NIGHTBUILD.md

**Audience:** the coding agent invoked to run a NightBuild for the user's current project.
**Canonical location:** `https://raw.githubusercontent.com/Stevenic/nightbuild/main/NIGHTBUILD.md`. Read it once at kickoff and keep it in conversation context for the rest of the run. **Pin the version at kickoff:** record the resolved commit SHA in `state.nightbuild_md_sha` (resolve `main` via `git ls-remote https://github.com/Stevenic/nightbuild.git HEAD`, taking the first 40 chars). On context loss, re-fetch from the SHA-pinned URL — `https://raw.githubusercontent.com/Stevenic/nightbuild/<sha>/NIGHTBUILD.md` — never from `main` again, so a mid-run upstream change cannot subtly shift behavior.

This file is self-contained — every scaffold the agent needs to write to the user's project is embedded below.

NightBuild is a pattern for handing off a coding task at bedtime and waking up to a working artifact plus a clean post-mortem. This file is the operational spec: when the user says "kick off a night build" — or you receive an Agent Ready prompt that names this URL — follow the kickoff flow below, then run the autonomous tick loop until success or a stop condition trips.

If this file and the README disagree, this file wins for behavior. The README explains the pattern for humans; this file specifies the contract for agents.

---

## Glossary

- **Setup** — first-time-on-this-repo bootstrap. One-shot. Creates `.nightbuild/`, writes the initial `TONIGHT.md`, configures `.gitignore`, ends with an overview. Does NOT start a run.
- **Kickoff** — the in-conversation entry into a single night's run. Agent-driven; the user only answers questions. Drains `TONIGHT.md`, writes the program, schedules the loop.
- **Tick** — one wakeup of the autonomous loop. Reads state, picks a sub-step, works, verifies, commits, reflects, schedules the next wakeup.
- **Run** — everything from kickoff through the morning hand-off. One run per night.
- **`run_dir`** — `.nightbuild/<YYYY-MM-DD>/` at the repo root, with `-HHMM` suffix on collision. Per-run scratch — `state.json`, `program.md`, `log.md`, optional `handoff.md`, raw outputs under `raw/`. Each run is fully self-contained inside its own `run_dir` — fresh dated folder = fresh state.
- **Program** — `<run_dir>/program.md`. The executable plan generated at kickoff that the loop runs each tick. Mission, phases with done-bars, scope deviations, authority overrides, budget.
- **Boot tick** — environment validation that runs in-conversation as the last step of kickoff (NOT as the first scheduled wakeup). Catches missing tools / failing prereqs while the user is still awake. If boot fails, surface immediately — do not schedule a wakeup.
- **Done-bar** — a testable assertion that proves a phase is complete without human judgement.
- **Morning-readiness gate** — the final smoke that proves the success criterion is met.
- **`.nightbuild/preferences.json`** — per-clone preferences (branch policy, notification command, cost-rate, etc.). Optional. Created lazily the first time the user picks an "always"/"never" answer or supplies a long-lived setting. **Always gitignored**, even when the user opts to track the rest of `.nightbuild/`, because the values are machine-specific (local notify commands, personal API rates) and should not propagate to other clones.
- **`<repo>/.nightbuild/learnings.md`** — *distilled* project-scoped knowledge base (flaky tests, slow steps, project conventions, gotchas). NOT a chronological log — the agent reads tonight's reflections, merges novel lessons into existing topic sections, bumps "last confirmed" dates on lessons reinforced, and refines or replaces entries that are sharpened. Read at kickoff so each run inherits what previous runs learned the hard way. Per-run reflections themselves live in `<run_dir>/log.md`. Tracked in **shared** mode (corpus follows the repo across clones); ignored in **private** mode (each driver's local).
- **`tick_in_progress`** — boolean flag in `state.json` set true at tick step 6, cleared at step 10. Lets the next tick detect that the prior tick crashed mid-work and recover deterministically rather than trusting stale state.

---

## Which flow to run

When the user invokes you with the Agent Ready prompt, decide:

- **`TONIGHT.md` does NOT exist at the repo root** → run **Setup flow**. End with the overview; do NOT chain into a run.
- **`TONIGHT.md` exists** → run **Kickoff flow**.
- **The user explicitly asks to "re-run setup" or similar** → run Setup flow even if `TONIGHT.md` exists (overwriting `.gitignore` and `TONIGHT.md` only with the user's confirmation).

---

## Setup flow

First-time bootstrap on a repo. One-shot — creates the files NightBuild needs, then hands the user an overview and exits without starting a run.

1. **Greet the user** with a one-line confirmation that you're setting up NightBuild on this project. Use the repo folder name if you can.

2. **Verify it's a git repo.** Run `git rev-parse --is-inside-work-tree`. If not a git repo, ask the user whether to `git init` here or stop — NightBuild commits each tick and requires git.

3. **Detect repo type.** Multi-person heuristic: `git log` shows ≥ 2 distinct authors, OR a `.gitignore` is already checked in. Otherwise solo.

4. **Gitignore configuration.** Ask one question:

   > Should this NightBuild be **private** (per-user — each driver maintains their own `TONIGHT.md` queue, run history, and `learnings.md` locally; nothing shared via git) or **shared** (project-wide — `TONIGHT.md`, run history, and `learnings.md` live in the repo and follow it across clones)?

   Recommend **shared** on a solo repo (single driver — keep the queue and learnings with the project history). Recommend **private** on a multi-person repo (avoids cross-driver merge conflicts and leakage of one driver's intent into another's run). State the recommendation and let the user override.

   Apply the answer by appending to `.gitignore` (creating the file if absent):

   - **Private** →
     ```
     .nightbuild/
     TONIGHT.md
     ```
   - **Shared** →
     ```
     .nightbuild/*/raw/
     .nightbuild/preferences.json
     ```
     `raw/` is large unreviewable build/test output; `preferences.json` is machine-specific (notify commands, API rates) and stays local in either mode.

   Skip lines that are already present.

5. **Create `.nightbuild/`** at the repo root (empty directory; per-run dated folders are created at kickoff).

6. **Write `TONIGHT.md`** at the repo root using the embedded scaffold from § Embedded scaffolds → "`TONIGHT.md` scaffold". Replace `<PROJECT NAME>` with the repo folder name.

7. **Show the overview** to the user (in your own voice, but covering all four points):

   > **NightBuild is set up.**
   >
   > **Across the day:** open `TONIGHT.md` and add tasks as ideas come up. Free-form is fine — focus, queued items, hints, hard nos.
   >
   > **At bedtime:** paste this prompt into me (or any coding agent) to start the night's run:
   >
   > ```
   > Read the NIGHTBUILD.md file at https://raw.githubusercontent.com/Stevenic/nightbuild/main/NIGHTBUILD.md
   > and follow its instructions to kick off a night build for this project.
   > ```
   >
   > I'll drain `TONIGHT.md`, ask a few clarifying questions to make each task testable (and on the first night, whether to put the run on a fresh `nightbuild/<date>` branch — easy rollback in the morning), write a per-run program under `.nightbuild/<YYYY-MM-DD>/program.md`, schedule the autonomous loop, and hand you a `MORNING_TODO.md` in the morning.
   >
   > **In the morning:** read `MORNING_TODO.md` at the repo root for the wake-up summary. The full per-iteration narrative lives in `<run_dir>/log.md`.
   >
   > **Optional:** configure notifications (Slack / OS notifications / email when the run finishes) and a cost-per-million-tokens rate by editing `.nightbuild/preferences.json`. Schema and examples are in NIGHTBUILD.md § "Embedded scaffolds → `.nightbuild/preferences.json`".

   Setup ends here. Do NOT proceed to kickoff in the same session unless the user explicitly asks.

---

## Kickoff flow

Precondition: setup has run on this repo (`TONIGHT.md` exists, `.nightbuild/` is gitignored). If `TONIGHT.md` is missing, run the Setup flow first instead of kickoff.

When invoked to start a NightBuild on the user's project:

1. **Read `TONIGHT.md`** at the repo root.
   - If queued tasks exist → ask clarifying questions one at a time until each task has a concrete acceptance criterion (artifact path + observable assertion). Stop the moment the spec is testable without further human input — over-questioning costs sleep.
   - If `TONIGHT.md` is empty (just the scaffold placeholders) → ask the user what to do tonight. Walk them through goal → success criterion → constraints, then proceed.

2. **Read `<repo>/.nightbuild/learnings.md` if it exists.** This is the project's distilled knowledge base — topic-organized lessons curated across past runs (flaky tests, slow steps, project conventions, gotchas). Read the whole file; it's intentionally kept small. Let it shape your clarifying questions and the program you'll write — e.g. if a known-flaky test is in the path tonight, plan to retry on first failure; if a known-slow step is queued, set its `delaySeconds` accordingly. Do not modify the file at kickoff; the End-of-overnight protocol distills tonight's reflections back into it.

3. **Pin the NIGHTBUILD.md version.** Resolve the canonical URL to a commit SHA: `git ls-remote https://github.com/Stevenic/nightbuild.git HEAD` (first 40 chars of the first line). Stash the SHA — you'll write it into `state.nightbuild_md_sha` at step 8. From this point on, any context-loss recovery re-fetches `https://raw.githubusercontent.com/Stevenic/nightbuild/<sha>/NIGHTBUILD.md`, never `main`, so a mid-run upstream change cannot subtly shift behavior.

4. **Optional cost cap.** Ask once:
   > Set a cost cap for tonight's run? (e.g. `$10`, `$25`, or skip — default is no cap).

   If the user gives a cap, capture it as a number into `state.inputs.budget_max_usd`. Also ask (or read from `.nightbuild/preferences.json` if `usd_per_mtok` is set) the user's per-million-token rate so the loop can estimate spend; if no rate is provided, just track raw token totals without dollar conversion. A saved-from-preferences rate is normal; users rarely change it.

5. **Branch decision.** Determine which branch this run will commit to. Resolve in order:
   - Read `.nightbuild/preferences.json` if it exists. If `branch_policy === "new"` → create a fresh branch and continue (no prompt). If `branch_policy === "current"` → stay on the current branch (no prompt).
   - Otherwise (no preferences file, or `branch_policy === "ask"`) → prompt the user:
     > Create a new branch for tonight's run? A fresh branch makes it trivial to roll back in the morning if you don't like the result — `git checkout <original>` and the night's commits are out of your way.
     >
     > **yes** (just tonight) / **no** (use current branch) / **always** (save and stop asking) / **never** (save and stop asking)
     
     If the answer is **always** or **never**, write `.nightbuild/preferences.json` with `branch_policy: "new"` or `"current"` respectively (schema in § Embedded scaffolds).
   - **If creating a new branch:** name it `nightbuild/<YYYY-MM-DD>` (suffix `-HHMM` on collision). Run `git checkout -b nightbuild/<YYYY-MM-DD>` from the current HEAD. Record the original branch in `state.inputs.parent_branch` so the morning hand-off can tell the user how to roll back or merge.
   - **If staying on the current branch:** if `git status --porcelain` shows uncommitted changes, pause and ask the user to commit or stash first — NightBuild's commits will otherwise mix with their in-flight work.

6. **Provision the run scratch.** Create `.nightbuild/<YYYY-MM-DD>/` (with `-HHMM` suffix if today's folder already exists). Initialize `<run_dir>/log.md` with a run header (date, pinned NIGHTBUILD.md SHA, kickoff Q&A summary, branch + parent_branch, cost cap if set).

7. **Write the program.** Generate `<run_dir>/program.md` using the schema in § Program format below. The program is what the loop executes — mission, phase plan with done-bars, scope deviations, authority overrides, inputs, budget.

8. **Seed `<run_dir>/state.json`** using the JSON in § Embedded scaffolds → "`state.json` seed" below. Fill in `run_dir`, captured inputs (`parent_branch` if branch-isolation was chosen, `budget_max_usd` and `usd_per_mtok` if applicable), `nightbuild_md_sha` (from step 3), and `started_at` (ISO timestamp).

9. **Drain `TONIGHT.md`.** Copy the pre-drain content verbatim into `<run_dir>/log.md` under a `## Kickoff inputs` section, then overwrite `TONIGHT.md` with the empty scaffold from § Embedded scaffolds → "`TONIGHT.md` scaffold" so it's ready for the next night's queue.

10. **Run the boot tick in-conversation.** See § Boot tick below — verify prereqs (tools installed, build currently passes if applicable, git healthy, program parseable). Do NOT schedule a wakeup yet. If anything fails, surface immediately so the user can fix it before going to bed; if all checks pass, advance `state.phase` from `"boot"` to the first real phase and tell the user "boot passed; scheduling first wakeup."

11. **Schedule the first wakeup** with `<<autonomous-loop-dynamic>>` as the prompt sentinel. `delaySeconds: 60` (the loop will pick up at the first real phase, since boot already ran). Then exit so the user can sleep.

The user does not write a `NIGHTBUILD.md` into their project. This file (the canonical version on GitHub) is the contract for every run.

---

## Boot tick

Runs in-conversation as kickoff step 10. Surfaces prereq failures while the user is still awake — never as a surprise at 7am. Do NOT write code in the boot tick; only verify.

1. **Tools.** For each tool the program will invoke (parse from `<run_dir>/program.md` — typical: `git`, `node`, `npm`/`pnpm`/`yarn`, `tsc`, `python`, `cargo`, the project's test runner), confirm it's on `PATH` and runs a `--version` cleanly.

2. **Build sanity.** If the project has a fast build/typecheck command, run it once to confirm the *starting* state is green. If it's already broken, the loop will spend the night chasing pre-existing failures — surface and stop.

3. **Git health.** `git status --porcelain` (already verified at branch decision, but re-confirm), `git rev-parse HEAD` (no detached-HEAD weirdness), and that the configured branch is what we expect.

4. **Program parseable.** Re-read `<run_dir>/program.md` and confirm it has at least Mission, one Phase with a done-bar, and Budget.

5. **Network reachability** *(only for projects whose phases require it)*. A 1-second `curl --head` of any required registry / API. If unreachable, ask the user to fix or stop.

If any check fails, write the failure mode to `<run_dir>/log.md` under `## Boot failed`, set `state.boot_error` to a one-line summary, surface to the user in chat, and exit kickoff WITHOUT scheduling a wakeup. The user fixes and re-kicks-off.

If all checks pass, append a `## Boot passed` block to `<run_dir>/log.md` with each check's outcome, set `state.phase` to the first real phase from the program's Phase plan, and proceed to kickoff step 11.

---

## Tick recipe

Every wakeup:

1. **Load context.** Read `<run_dir>/state.json` and `<run_dir>/program.md`. NIGHTBUILD.md was loaded at kickoff and remains in conversation context — if it's missing (context reset), re-fetch from the SHA-pinned URL `https://raw.githubusercontent.com/Stevenic/nightbuild/<state.nightbuild_md_sha>/NIGHTBUILD.md`, never `main`.

2. **Crash-recovery check.** If `state.tick_in_progress === true`, the previous tick crashed mid-work. Reconcile:
   - Compare `state.head_at_tick_start` to `git rev-parse HEAD`. If HEAD has new commits beyond the snapshot, the previous tick committed but didn't finish housekeeping — accept the commit(s), advance `iteration` and `last_completed_step` from the latest commit message, append a `## Tick recovered (post-commit crash)` entry to `<run_dir>/log.md`.
   - If HEAD is unchanged, the previous tick crashed before committing — discard any uncommitted working-tree changes (`git restore --staged --worktree :/`) and retry the same step. Append `## Tick recovered (pre-commit crash)`.
   - In both cases, set `tick_in_progress = false` before continuing.

3. Check stop conditions (§ Stop conditions). If tripped, write the final entry, exit without scheduling.
4. Pick the next sub-step within the current phase. Keep sub-steps under ~20 min of wall-clock work.
5. Read recent reflections (last 3) from `<run_dir>/log.md`. If a recurring blocker pattern is visible, incorporate the lesson into this step.
6. **Begin work.** Set `state.tick_in_progress = true`, `state.head_at_tick_start = <current HEAD SHA>`, write state. Then do the work. No exploration of unrelated parts of the repo.
7. Verify: build, unit test, smoke for the step. Stream raw output to `<run_dir>/raw/<phase>-<iteration>.log` so context stays small. If failed, iterate up to 3 times within this tick. After 3 failures, write a `BLOCKED` entry and yield without advancing state. Increment `consecutive_blocked`.
8. Commit. `git add -A && git commit -m "<phase>: <one-line summary>"`. Never bypass hooks. Never `--amend`. Never push.
9. Reflect (see § Reflection structure). Append to `<run_dir>/log.md`.
10. **Finish housekeeping.** Update state: increment `iteration`, set `last_completed_step`, append a history entry, reset or increment `consecutive_blocked`, **add this tick's token counts to `state.tokens_total` (and `state.usd_estimate` if `inputs.usd_per_mtok` is set)**, set `tick_in_progress = false`, persist.
11. Append an iteration log section to `<run_dir>/log.md` — commands run, state delta, decisions logged, next step.
12. Schedule the next wakeup with appropriate `delaySeconds` (§ Wakeup pacing).

---

## Reflection structure

Every tick ends with five short lines appended to `<run_dir>/log.md`:

```markdown
### Reflection — iteration N

**What I did:** one sentence.
**What worked:** one bullet — concrete win to repeat.
**What didn't:** one bullet — concrete miss to avoid; null if nothing notable.
**Improvement for next pass:** one bullet — actionable change to the next tick's approach.
**Confidence in current phase done-bar:** low / medium / high, with one-line justification.
```

Long reflections waste context. The "improvement for next pass" bullet is the part future ticks read in step 4 — keep it actionable. Pure-correction reflections push the loop toward overcaution; recording successes preserves validated approaches.

---

## Wakeup pacing

The Anthropic prompt cache has a 5-minute TTL. Sleeping past 300 seconds means the next wakeup reads the conversation context uncached.

- **Active phases (compile errors, fast feedback):** `delaySeconds: 60`–`120`. Cache stays warm.
- **Longer-running steps (installer build, full smoke):** `delaySeconds: 120`–`270`. Still in cache.
- **BLOCKED + need diagnostic time:** `delaySeconds: 270`.
- **Idle waiting on something specific:** `1200`–`1800` (one cache miss buys a longer wait).

Never `300` (worst-of-both). Never past `1800` for an active project.

---

## Stop conditions

Stop and do not reschedule when ANY of these is true:

- All phase done-bars are green AND the morning-readiness gate passes.
- `consecutive_blocked >= 3`. Write `STOPPED: stuck on <step>` and a top-of-log `## NEEDS HUMAN` block describing the blocker.
- Wall clock since iteration 1 exceeds the wall-clock budget cap (default 6 hours; the program may override).
- **Cost cap exceeded** *(only if `state.inputs.budget_max_usd` is set)*: `state.usd_estimate >= state.inputs.budget_max_usd`. Write `STOPPED: cost cap reached at $<usd_estimate>` and a top-of-log `## NEEDS HUMAN` block.
- A required input is missing from state.
- The user creates a `STOP` sentinel file at the repo root.
- The branch has uncommitted changes the loop didn't make (defensive — pause rather than overwrite).

On success, write a top-of-log `## MORNING SUMMARY` to `<run_dir>/log.md`. On block, write `## NEEDS HUMAN`. Do this *immediately* on hitting the criterion, not at the end of the run.

---

## Authority and boundaries

Default allow-list:

- Edit files inside the project.
- Run language/framework tooling locally (`npm`, `tsc`, language-specific build).
- Run unit + smoke tests locally.
- Install language packages from public registries.
- `git init`, `git add`, `git commit` (in the project).
- `git checkout -b nightbuild/<YYYY-MM-DD>` at kickoff if the user opts in to branch isolation. The loop never switches branches mid-run.

Default deny-list — require explicit opt-in via the program's "Authority overrides" section:

- Push to any remote (`git push`, `gh pr create`, equivalent).
- Force-push.
- Delete files outside the project.
- Skip git hooks (`--no-verify`).
- Disable signing or security baselines.
- Modify global git config or shell profile.
- Touch external-account state (Stripe products, OAuth client IDs, cloud resources).

If the loop encounters something it would normally need to bypass (a failing pre-commit hook, a missing out-of-project dependency), STOP and surface the blocker. Do not "make it work" by disabling safeguards.

---

## Logging hygiene

- Never log secrets, API keys, tokens, or paths inside identity stores.
- Truncate test/build output longer than 20 lines in `log.md`: keep first/last 5 with `... <N lines elided> ...` between. Full output goes to `<run_dir>/raw/`.
- Commit messages: `phase: short summary`. No emojis.
- Append-only on `<run_dir>/log.md` — do not edit prior entries (the post-mortem narrative is built from them).

---

## End-of-overnight protocol

When the success criterion is met (or the run terminates on a stop condition), produce:

1. **`<run_dir>/handoff.md`** — pre-composed text the user can pipe directly into `git commit -F` or `gh pr create --body-file`. Format:

   ```markdown
   <commit-subject one-liner, ≤72 chars, no trailing period>

   <2–4 sentence summary of what landed and what's left>

   ## What changed
   - <bullet — one line per logical change>

   ## What didn't land / known issues
   - <bullet, or "none">

   ## How to validate
   ```bash
   <commands a reviewer should run>
   ```
   ```

   Line 1 is the commit subject *and* the suggested PR title. Everything below it is the body / PR description. Skip this file only if the run made no commits.

2. **`MORNING_TODO.md`** at the repo root — wake-up summary. Required sections:
   - **Status** — success / blocked / partial.
   - **What landed** — concrete artifacts produced, with paths.
   - **What's blocked / needs human** — only on block.
   - **Commit & PR** — *(only when in a git repo)*. Concrete `git` commands the user can run as-is. Branch in two ways:
     - **Branch-isolation runs** (a `nightbuild/<YYYY-MM-DD>` branch was created at kickoff): give three options — keep + merge (`git checkout <parent_branch> && git merge nightbuild/<YYYY-MM-DD>`), keep + open a PR (`git push -u origin nightbuild/<YYYY-MM-DD> && gh pr create --title "$(head -n1 <run_dir>/handoff.md)" --body-file <run_dir>/handoff.md`), or discard (`git checkout <parent_branch> && git branch -D nightbuild/<YYYY-MM-DD>`). Show the diff command up front: `git log <parent_branch>..HEAD --oneline`.
     - **Current-branch runs** (the user opted to commit to their existing branch): give two options — push as-is (`git push`) or open a PR (`gh pr create --title "$(head -n1 <run_dir>/handoff.md)" --body-file <run_dir>/handoff.md`). Note that the loop already committed each tick, so there is nothing to commit further unless the loop ended in BLOCKED with uncommitted changes — in that case, list the dirty paths and a suggested final commit command.
   - **Resume instructions** — only on block. How to pick up where the loop left off.

   Always link to `<run_dir>/handoff.md` in the Commit & PR section so the user knows the message text is pre-composed.

3. **Top-of-log block** in `<run_dir>/log.md` — `## MORNING SUMMARY` (success) or `## NEEDS HUMAN` (blocked).

4. **Distill into `<repo>/.nightbuild/learnings.md`** *(create if absent)*. Read the existing file (if any) and tonight's reflections from `<run_dir>/log.md`, then update learnings.md so it remains a *distilled* knowledge base organized by topic — NOT a chronological log of every run's reflections. The per-run narrative already lives in `<run_dir>/log.md`; learnings.md is for lessons future runs should inherit.

   Operations on each lesson surfaced from tonight's reflections:
   - **Add** a new bullet to the appropriate section if the lesson is novel (not already present).
   - **Bump** the `last confirmed` date on an existing entry if tonight's run hit the same lesson again — do not duplicate the entry.
   - **Refine or replace** an existing entry if tonight's run sharpens, narrows, or contradicts it.
   - **Skip silently** if tonight's run produced no novel or confirming lessons. Uneventful runs add nothing.
   
   Keep the file small and scannable. Format in § Embedded scaffolds → "`learnings.md` format" — topic sections (Flaky / unreliable, Slow steps, Project conventions, Gotchas, Deferred / open), each entry a single bullet with provenance dates. The agent can add new sections as they're warranted. If learnings.md grows past ~100 lines, prefer pruning over appending — but leave deletions to the user unless an entry has been clearly contradicted.

5. **Fire the notify hook** *(only if `.nightbuild/preferences.json` has a `notify` command)*. Spawn the command with these env vars set: `NIGHTBUILD_STATUS` (`success` | `blocked`), `NIGHTBUILD_SUMMARY` (one-line subject from `<run_dir>/handoff.md` line 1, or the BLOCKED reason), `NIGHTBUILD_RUN_DIR` (absolute path), `NIGHTBUILD_REPO` (repo folder name). Run it, log stdout/stderr to `<run_dir>/raw/notify.log`, ignore non-zero exit (don't block the run on a flaky webhook). If no `notify` is configured, skip silently.

6. **Clean git state.** No uncommitted changes left from the loop's work.

Write the morning artifacts *immediately* on hitting the success criterion — before any stretch work. A run that hits success and then runs out of budget on stretch without writing the morning hand-off is strictly worse than one that exits cleanly at success.

---

## Program format

`<run_dir>/program.md` is generated at kickoff (step 3) and read on every tick. It's the program the loop executes. Schema:

````markdown
# Program — <YYYY-MM-DD>

## Mission

<One or two sentences. Concrete artifact path + observable assertion. Example: "An installable .exe under <repo>/dist-installer/ that opens a window, performs <X>, and a smoke-test script proves the round trip without human interaction.">

## Scope deviations

| Spec requirement | Substitution | Reason |
|---|---|---|
| <real thing> | <stub thing> | <constraint that forced this> |

**Architectural invariants retained:** <list things that must remain true even with substitutions>.

## Inputs (from kickoff Q&A)

- `<input_1>` — <value>
- `<input_2>` — <value>

## Phase plan

### Phase A — `<name>`

<bullet list of work>

**Done bar:** <observable assertion, ideally a smoke that exits 0>.

### Phase B — `<name>`

...

### Phase F — End-to-end smoke (the morning-readiness gate)

**Done bar:** `<command>` exits 0 on a fresh state.

### Phase G — `polish` *(optional)*

Only attempt if A–F are green AND budget remains.

## Authority overrides

Anything that diverges from NIGHTBUILD.md defaults — e.g. "this project allows `git push` to a feature branch", or "the pre-commit hook requires `npm run format` before commit and that is allowed".

## Budget

- Cap: <wall-clock hours, default 6>
- Started at: <ISO timestamp>
````

Keep the program tight. Don't add sections beyond what the loop needs to act.

---

## Embedded scaffolds

Files the agent writes into (or appends-to under) the user's project. Schemas below are canonical — copy them verbatim and fill in the placeholders.

### `TONIGHT.md` scaffold

Written at kickoff step 6 (drain). Replace `<PROJECT NAME>` with the actual project name; leave the section bodies as the example placeholders so the user has hints next time.

````markdown
# TONIGHT.md — <PROJECT NAME>

**Role:** the queue of things you want NightBuild to do the next time you kick off a run.
**You write to this file.** The agent drains it at kickoff (synthesizes it into the per-run program at `<run_dir>/program.md`, asks you clarifying questions, then resets this file to the empty scaffold). The autonomous loop does NOT read this file during ticks.

> Add to this file across the day as ideas come up. When you kick off a night build, the agent reads what's here, asks the questions it needs to make each task testable, writes the program, and clears this queue. Whatever was here gets archived verbatim in `<run_dir>/log.md` under "Kickoff inputs".
>
> If this file is empty when you kick off, the agent will ask you what to do tonight from scratch.

---

## Focus for the next run

<one or two sentences: what should the run bias toward? e.g. "land Phase D — the smoke is the bottleneck", or "stretch goal: try the Windows installer path if A–F land before 2am">

## Tasks queued

- [ ] <task 1 — concrete, scoped, single phase if possible>
- [ ] <task 2>
- [ ] <task 3>

## Hints / context that won't be obvious from the repo

- <e.g. "the test in foo_test.go is flaky on Windows — retry once before treating as a real failure">
- <e.g. "I rebased main this evening; the branch is at <sha>">

## Hard nos for the run

- <e.g. "do not touch the auth middleware — I'm in the middle of refactoring it">
- <e.g. "no dependency upgrades">
````

### `state.json` seed

Written at kickoff step 8 to `<run_dir>/state.json`. Lives inside the run folder so each run is fully self-contained — there is no state file at the repo root. Fill in `run_dir`, `started_at`, captured `inputs` from the kickoff Q&A, `nightbuild_md_sha`, and the real phase names in `todo` from the program. Leave `boot_error`, `last_completed_step`, etc. as `null` / empty — the loop sets them.

```json
{
  "schema_version": 4,
  "phase": "boot",
  "iteration": 0,
  "last_completed_step": null,
  "consecutive_blocked": 0,
  "boot_error": null,
  "started_at": null,
  "run_dir": null,
  "nightbuild_md_sha": null,
  "tick_in_progress": false,
  "head_at_tick_start": null,
  "tokens_total": 0,
  "usd_estimate": 0.0,
  "morning_ready": false,
  "morning_ready_at": null,
  "stretch_completed": [],
  "inputs": {},
  "secrets": {},
  "todo": ["boot", "<phase-a>", "<phase-b>", "<phase-c>", "<phase-d>", "<phase-e>", "<phase-f>", "<phase-g-optional>", "<phase-h-optional>"],
  "phases_done": [],
  "deliverables": {},
  "history": []
}
```

Field notes:

- `run_dir` — set at kickoff to `.nightbuild/<YYYY-MM-DD>/` (with `-HHMM` suffix on collision). Where this run's `program.md`, `log.md`, and `raw/` artifacts live.
- `nightbuild_md_sha` — the canonical NIGHTBUILD.md commit SHA pinned at kickoff. On context loss, the recovery agent fetches `https://raw.githubusercontent.com/Stevenic/nightbuild/<sha>/NIGHTBUILD.md` to load the *exact* version the run started under, never `main`.
- `tick_in_progress` — set true at tick step 6, false at step 10. If a tick reads it as true, the previous tick crashed mid-work; the recovery sequence in tick step 2 reconciles state vs `git log`.
- `head_at_tick_start` — `git rev-parse HEAD` snapshot at tick step 6, used by recovery to determine whether the prior tick committed before crashing.
- `tokens_total` — cumulative input+output tokens consumed across all ticks. Always tracked.
- `usd_estimate` — cumulative dollar estimate, computed as `tokens_total * inputs.usd_per_mtok / 1_000_000`. Stays at `0.0` if no rate is set.
- `inputs` — captured Q&A answers from kickoff that the loop must NOT lose (paths, flags, environment-specific knobs). Common fields: `inputs.parent_branch` (when branch isolation was chosen), `inputs.budget_max_usd` (optional cost cap), `inputs.usd_per_mtok` (optional cost-per-million-tokens rate, copied from `preferences.json` if set there).
- `secrets` — never committed; populated only if the program declares secret inputs. Treat as opaque; never log or echo.
- `todo` — replace placeholders with the real phase names from the program. The agent advances through this list.

Gitignore configuration is owned by the Setup flow and persisted in the project's `.gitignore` itself — there's no redundant copy in state. Branch policy, notification command, and per-token rate are owned by `.nightbuild/preferences.json` (see below).

### `.nightbuild/preferences.json` *(optional)*

Created lazily the first time the user picks an "always"/"never" answer at any prompt or supplies a long-lived setting (cost rate, notification command). Read at kickoff. Always gitignored — even when the user opts to track the rest of `.nightbuild/`, this file stays local because its values (notify URLs/commands, personal API rates) are machine-specific and shouldn't propagate to other clones.

```json
{
  "schema_version": 2,
  "branch_policy": "new",
  "notify": "curl -s -X POST -H 'Content-type: application/json' -d \"{\\\"text\\\":\\\"NightBuild $NIGHTBUILD_STATUS in $NIGHTBUILD_REPO: $NIGHTBUILD_SUMMARY\\\"}\" https://hooks.slack.com/services/...",
  "usd_per_mtok": 15.0
}
```

Field notes:

- `branch_policy` — one of `"new"` (always create a fresh `nightbuild/<YYYY-MM-DD>` branch at kickoff), `"current"` (always commit to the current branch), or `"ask"` (prompt each time; equivalent to no preference). Set by the user picking "always" or "never" at the kickoff Branch decision prompt. Absent file = same as `"ask"`.
- `notify` — optional shell command spawned by End-of-overnight protocol step 5 with these env vars set: `NIGHTBUILD_STATUS` (`success`|`blocked`), `NIGHTBUILD_SUMMARY` (one-line subject), `NIGHTBUILD_RUN_DIR` (absolute path to the run folder), `NIGHTBUILD_REPO` (repo folder name). Common shapes:
  - **Slack:** `curl -s -X POST -H 'Content-type: application/json' -d "{\"text\":\"NightBuild $NIGHTBUILD_STATUS: $NIGHTBUILD_SUMMARY\"}" https://hooks.slack.com/services/...`
  - **macOS notification:** `osascript -e "display notification \"$NIGHTBUILD_SUMMARY\" with title \"NightBuild $NIGHTBUILD_STATUS\""`
  - **Linux notification:** `notify-send "NightBuild $NIGHTBUILD_STATUS" "$NIGHTBUILD_SUMMARY"`
  - **Email:** `echo "$NIGHTBUILD_SUMMARY" | mailx -s "NightBuild $NIGHTBUILD_STATUS" me@example.com`
  
  Treat the value as opaque shell — quote env vars correctly. Non-zero exit is logged but does not block the run.

- `usd_per_mtok` — optional dollars-per-million-tokens rate used to compute `state.usd_estimate` and to enforce `state.inputs.budget_max_usd`. Set once (the user's API rate rarely changes). If absent, the loop tracks raw token totals without a dollar conversion.

If the user later wants to change a saved preference, they can delete or edit `.nightbuild/preferences.json` directly — the agent reads it fresh at every kickoff. Bump `schema_version` when the shape changes.

### `<repo>/.nightbuild/learnings.md` format

A distilled, topic-organized knowledge base. **Not** a per-run journal — the per-run narrative lives in each `<run_dir>/log.md`. Read at kickoff step 2 so the agent inherits past lessons; updated by End-of-overnight step 4 (the agent merges, refines, and bumps confirmation dates rather than appending fresh blocks each run).

```markdown
# Learnings — <project name>

Distilled lessons from past NightBuild runs on this project. Future kickoffs read this file to anticipate gotchas. Sections are organized by topic; the agent prunes, merges, and refines entries to keep the file small. Per-run reflection history lives in each run's `<run_dir>/log.md`.

---

## Flaky / unreliable

- `<test path or step>` — what fails, how often, what to try first. *first seen YYYY-MM-DD; last confirmed YYYY-MM-DD*

## Slow steps

- `<command or phase>` — observed wall-clock and pacing rule (e.g. `delaySeconds` choice, skip-when-engine-untouched). *first seen YYYY-MM-DD*

## Project conventions

- <convention or constraint that wasn't obvious from a fresh read of the repo — package manager choice, hook requirements, naming rules>. *first seen YYYY-MM-DD*

## Gotchas

- <surprise that bit a previous run; how to avoid it next time>. *first seen YYYY-MM-DD; last confirmed YYYY-MM-DD*

## Deferred / open

- <work a previous run deliberately punted on; may be worth a future TONIGHT.md item>. *deferred YYYY-MM-DD in run <run_dir>*
```

Entry shape rules:

- Each entry is a single bullet. If a lesson grows past two lines, it's probably two lessons — split.
- Provenance dates: `first seen` is when the lesson was added; `last confirmed` is the most recent run that hit it again. Both are optional but useful for spotting stale entries.
- Sections are flexible — the agent can add new ones (e.g. "CI quirks", "External dependencies") if a category recurs. Avoid one-off sections.
- A run that learns nothing new touches nothing. Uneventful runs do not show up here.
- The user can edit/prune freely — the agent reads whatever shape it finds.
