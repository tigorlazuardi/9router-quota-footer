# Ralph Contract — quota/001-9router-quota-plugin

## Orchestrator model — check FIRST, before anything else

This contract is authored for an orchestrator running as **`sonnet`** (Opus chose this when planning; it is also recorded in RALPH_PROGRESS.md). Before reading tasks or spawning anything, confirm your own model matches.

Look at your model identity in system context. If it does **not** match `sonnet`, STOP immediately and output exactly this, then end the turn:

> ⛔ Wrong orchestrator model. This ralph contract requires **`sonnet`**, but you are running a different model. Switch with `Ctrl+P` → `sonnet`, then `/clear` (or a fresh session) and restart the loop with the same start command. This guard exists because it is easy to forget to set the model, and running the loop on the wrong one silently over-spends or under-performs.

This is an authorized early exit (like the abort protocol) — it is **not** a false finish, so do **not** emit the completion promise. Only proceed past this check when your model matches `sonnet`.

You are the **`sonnet` orchestrator** for this slice. You drive the work but you do **not** do heavy implementation yourself — you delegate to subagents and keep the ledger. Re-read this contract and `RALPH_PROGRESS.md` at the start of every iteration and act on whatever is incomplete.

**Caveman output (default).** Operate caveman-compressed: drop articles/filler/pleasantries/hedging, fragments OK, keep all technical substance — narration is the cheapest thing to cut. Tell every subagent you spawn to report caveman too. Stay normal for: code, commit messages, `RALPH_PROGRESS.md` content, the `<promise>` tag, and anything security-related.

## Completion promise (how this loop exits)

Output `<promise>RALPH SLICE FINISHED</promise>` **only** when one of these is unequivocally true:

- **Success:** every task below is `done` and reviewed, the integration branch is green (`bunx tsc --noEmit && bunx vitest run` passes), everything is committed, and the merge-handoff summary is written to RALPH_PROGRESS.md.
- **Abort:** an Opus circuit-breaker decision (see below) returned `ABORT`, and you have recorded the reason in RALPH_PROGRESS.md.

Both are genuine terminal states, so emitting the promise is honest in both. Never emit it to escape a hard iteration — if you're stuck but not terminal, keep working or trigger the circuit-breaker.

**Echo guard.** The literal tag `<promise>RALPH SLICE FINISHED</promise>` ends the loop the instant it appears in your output — the stop hook scans your last message for it. So **never type that tag** except as the final line of a true terminal state. Everywhere else (notes, plans, RALPH_PROGRESS.md, talking about it) call it "the promise"; do not write the tag itself. Accidentally echoing it = false finish.

## Pre-flight sanity check (first iteration only; re-verify cheaply after)

Before any task work, confirm the ground is solid. If any check fails and you cannot fix it quickly, record it in RALPH_PROGRESS.md and treat it as an abort condition.

- [ ] Base branch `main` is checked out and clean.
- [ ] Required tooling present: `bun`, `git`. (`bunx tsc` / `bunx vitest` resolve after B1 installs devDeps.)
- [ ] Integration branch `ralph/quota-plugin` exists (create from `main` if not).
- [ ] Baseline truth signal: `bunx tsc --noEmit && bunx vitest run` is green. NOTE: repo is greenfield — this command does NOT pass until task **B1 (bootstrap)** lands package.json + tsconfig + vitest + a smoke test. B1 is the task that MAKES the baseline green; treat B1 as the bootstrap that establishes the truth signal, then every later batch must keep it green.
- [ ] Assumptions still hold: pi loads `.ts` extensions directly (no bundler); `@earendil-works/pi-coding-agent` is a peerDependency (`"*"`), not bundled; 9router runs at resolved baseUrl with security OFF (no auth headers).

## Implementation strategy

Build a standalone pi extension package (TypeScript) implementing TWO features from `PLAN.md`: (1) a 9router quota footer via an adapter pattern, and (2) a rate-limit wakeup hook. Order of attack: bootstrap the toolchain first (B1) so the truth signal exists, then land shared foundation (config + adapter types) in parallel (B2), then the two independent logic units — Claude-Code adapter and wakeup — in parallel (B3), then wire everything into `index.ts` (B4). Design decisions already fixed by the planner and NOT to be re-litigated: adapter OWNS the footer sentence (plugin holds zero Claude-Code wording); footer visible iff `event.model.provider === "9router"`; adapter chosen by model-id prefix (`cc/` → ClaudeCode); wakeup uses `after_provider_response` 429 + `retry-after` (primary) with quota `resetAt` fallback, a single replaceable `setTimeout`, and `pi.sendUserMessage("rate limit reset, lanjutkan task sebelumnya")` to resume the SAME session. Keep pure logic (format, countdown, retry-after parse, delay calc, throttle) in small testable functions; unit-test them with vitest. Mock fetch/hook in tests — do NOT hit a live 9router. Do NOT touch the vendor `pi-9router-ext` package or import it. Do NOT add a bundler (esbuild/rollup) — pi loads `.ts` directly. Keep all debug logging gated behind `NINE_ROUTER_QUOTA_DEBUG=1`.

## State protocol (resume-safe)

`RALPH_PROGRESS.md` is the source of truth for progress. On every iteration: read it, find the first incomplete batch, and continue. After **each** task reaches a terminal status, update RALPH_PROGRESS.md and **commit it inside the integration branch** so a crash/power-loss/stop can resume exactly here. Never redo a task already marked `done`.

**Resume reconciliation.** A crash can leave a worktree with uncommitted WIP that RALPH_PROGRESS.md doesn't know about. On every iteration start, before trusting the ledger: `git status` each live worktree. Uncommitted changes → inspect, then commit as WIP or stash, and reconcile the task's real status with RALPH_PROGRESS.md. Don't restart a task that's actually half-done on disk.

**Heartbeat.** Each iteration, append one line to RALPH_PROGRESS.md's progress log: `iteration N · <what advanced>`. Lets a human `tail` progress and powers no-progress detection below.

## No-progress guard (stuck = stop, don't burn tokens)

Track `last-progress-iteration` in RALPH_PROGRESS.md — the last iteration a task reached `done`. If the current iteration exceeds it by **3** with no task advancing, you're spinning: trigger the Opus circuit-breaker (below) on the blocking task instead of looping further. Cheaper to ask once than spin ten times.

## Worktree model

- Integration branch for this slice: `ralph/quota-plugin` (off `main`).
- **Parallel batch:** run each task in its own worktree, e.g. `git worktree add ../9router-quota-footer-<task-id> ralph/quota-plugin`. After review passes, merge that worktree into `ralph/quota-plugin`, resolve conflicts, then `git worktree remove` it.
- **Sequential/single task:** work directly on `ralph/quota-plugin`; no extra worktree needed.

Pass the **absolute worktree path** to every subagent and tell it to `cd` there first — subagents do not inherit your working directory.

**Concurrency cap: 4.** Never run more than **4** worktrees/implementers at once. Batch bigger than 4 → split into waves of ≤4. (No batch here exceeds 2.)

**Worktree is not a full env.** `git worktree add` skips gitignored files — no `node_modules`. A fresh worktree will fail type-check/test and look like a task failure. So after creating each worktree, before any implementer touches it: run the install command. Record `<install-cmd>` once here: `bun install`. No `.env` / `.envrc` / direnv in this repo — nothing else to copy.

**Parallel isolation.** No ports/DB/servers in this package — all tests are pure in-process unit tests with mocked fetch/timers. Isolation scheme: **none needed** (each worktree is a separate checkout; vitest runs in-process, no shared external resource). Give each worktree only its own directory.

## Tasks (batches run in order; tasks within a batch run in parallel)

### Batch B1 (bootstrap — establishes the truth signal)
- **b1-bootstrap — package + toolchain + smoke test** · tier `S` · reviewer `sonnet` · worktree `no`
  - Do: create `package.json` (name `pi-9router-quota`, type `module`, a `pi` manifest declaring the extension entry `src/index.ts` per pi packages docs, `peerDependencies: { "@earendil-works/pi-coding-agent": "*" }`, devDeps `typescript` + `vitest`, scripts `typecheck: "tsc --noEmit"` and `test: "vitest run"`); `tsconfig.json` (strict, `moduleResolution: bundler` or `node16`, `noEmit`, target ES2022, include `src`); `vitest.config.ts` (node environment); `.gitignore` (`node_modules`, build caches); a placeholder `src/index.ts` exporting a no-op default `(pi) => {}` extension factory so tsc has an entry; and a smoke test `src/__tests__/smoke.test.ts` asserting `1+1===2`. Run `bun install`.
  - Done when: `bunx tsc --noEmit && bunx vitest run` exits 0 on a clean tree (smoke test passes, no type errors). This is the baseline truth signal for all later batches.
  - Touches: `package.json`, `tsconfig.json`, `vitest.config.ts`, `.gitignore`, `src/index.ts`, `src/__tests__/smoke.test.ts`

### Batch B2 (depends on B1) — shared foundation, parallel
- **b2-config — baseUrl resolver** · tier `S` · reviewer `sonnet` · worktree `yes`
  - Do: `src/config.ts` — `resolveBaseUrl()` returning env `NINE_ROUTER_BASE_URL`, else `~/.pi/agent/9router-config.json` `baseUrl`, else `http://localhost:20128`. Read the file defensively (missing/malformed → fall through, never throw). Also export a small debug logger gated by `NINE_ROUTER_QUOTA_DEBUG=1`. Do NOT import the vendor `pi-9router-ext`.
  - Done when: unit tests cover all three resolution branches (env set; env unset + valid config file; neither → default) and the malformed-file fallthrough, using a mocked/temp home path or injectable readers; `bunx tsc --noEmit && bunx vitest run` green.
  - Touches: `src/config.ts`, `src/__tests__/config.test.ts`
- **b2-types — adapter contract + quota types** · tier `S` · reviewer `sonnet` · worktree `yes`
  - Do: `src/adapters/types.ts` — `QuotaAdapter` interface (`prefix: string`, `connectionProvider: string`, `format(quota: QuotaResponse): string | undefined`) and `QuotaResponse` types matching the `/api/usage` JSON in `PLAN.md` (`plan`, `quotas` keyed by `"session (5h)"` / `"weekly (7d)"` / `"weekly sonnet (7d)"` each with `used,total,remaining,remainingPercentage,resetAt,unlimited?`).
  - Done when: types compile under strict tsc; a trivial type-level or runtime test imports them and constructs a sample `QuotaResponse`; `bunx tsc --noEmit && bunx vitest run` green.
  - Touches: `src/adapters/types.ts`, `src/__tests__/types.test.ts`

### Batch B3 (depends on B2) — feature logic, parallel
- **b3-cc-adapter — Claude Code footer adapter** · tier `M` · reviewer `sonnet` · worktree `yes`
  - Do: `src/adapters/claude-code.ts` — `claudeCodeAdapter: QuotaAdapter` with `prefix: "cc/"`, `connectionProvider: "claude"`. `format()` reads ONLY `quotas["session (5h)"]` and `quotas["weekly (7d)"]` (ignore `weekly sonnet (7d)`), builds EXACTLY `cc {sessionUsedPct}% · session {sessionCountdown} · weekly {weeklyUsedPct}% · {weeklyCountdown}` (e.g. `cc 52% · session 1h41m · weekly 20% · 2d14h`). `usedPct = round(100 * used / total)`. Countdown computed from `resetAt` ISO vs now → compact `1h41m` / `2d14h` (helper `formatCountdown(msRemaining)`). Missing/`unlimited` window → omit that segment gracefully. Pass a `now` param (or injectable clock) so tests are deterministic.
  - Done when: unit tests cover the exact happy-string, used-% rounding when total≠100, countdown formatting (`1h41m`, `2d14h`, sub-hour, expired→0), `weekly sonnet` ignored, and graceful omission when a window is missing/unlimited; `bunx tsc --noEmit && bunx vitest run` green.
  - Touches: `src/adapters/claude-code.ts`, `src/__tests__/claude-code.test.ts`
- **b3-wakeup — rate-limit wakeup logic** · tier `M` · reviewer `sonnet` · worktree `yes`
  - Do: `src/wakeup.ts` — pure helpers + a `WakeupScheduler` class/factory that is hook-agnostic (takes an injected `sendUserMessage`, `now()`, and `setTimeout/clearTimeout`, so it's fully unit-testable). Helpers: `parseRetryAfter(headerValue, now): number|undefined` handling BOTH integer-seconds and HTTP-date forms (RFC 7231), returning ms delay; `computeDelay({retryAfter, resetAt}, now): number|undefined` — prefer retry-after, fall back to nearest `resetAt`, clamp negatives/absurd to undefined. Scheduler: `onRateLimited({retryAfter, resetAt})` schedules a single timer (replacing any existing), and on fire calls `sendUserMessage("rate limit reset, lanjutkan task sebelumnya")`; `cancel()` clears it (used when user resumes manually). Debug-log via the config logger.
  - Done when: unit tests (fake timers) cover: schedule from retry-after seconds; schedule from HTTP-date; fallback to resetAt when header absent; second call replaces timer (only one fires); cancel prevents fire; negative/absurd delay ignored; on fire `sendUserMessage` called exactly once with the exact string; `bunx tsc --noEmit && bunx vitest run` green.
  - Touches: `src/wakeup.ts`, `src/__tests__/wakeup.test.ts`

### Batch B4 (depends on B3) — integration wiring
- **b4-index — plugin wiring (footer + wakeup hooks)** · tier `M` · reviewer `sonnet` · worktree `no`
  - Do: replace the placeholder `src/index.ts` with the real extension factory. Wire: adapter registry `[claudeCodeAdapter]`; on `session_start` + `model_select`, detect active model — if `provider !== "9router"` → `ctx.ui.setStatus("9router-quota", undefined)` (hide); else pick adapter by id-prefix, resolve connection id via `GET /api/providers/client` (cached per session, re-resolve on 404), fetch `GET /api/usage/{id}` THROTTLED (~10–15s; reuse last quota within window; recompute countdown locally each render), then `ctx.ui.setStatus("9router-quota", adapter.format(quota))`. Refresh triggers: `session_start`, `model_select` (9router only), and a turn boundary (`turn_end`). Fetch errors/offline → never throw into the footer (hide or quiet `—`) and debug-log the error+status. Wire wakeup: `after_provider_response` → on `event.status === 429` build `{retryAfter: event.headers["retry-after"], resetAt: nearest cached quota resetAt}` and call the `WakeupScheduler`; cancel the pending wakeup on `turn_start`/user activity to avoid double-trigger. All debug-gated logging via config logger.
  - Done when: `bunx tsc --noEmit` passes (whole package type-checks with real wiring); existing unit tests still green; add focused tests for the wiring seams that are pure (adapter-selection-by-prefix, provider-hide decision, throttle gate) with fetch/ctx mocked — do NOT require a live 9router or live pi session; `bunx tsc --noEmit && bunx vitest run` green. Manual note in RALPH_PROGRESS.md: which PLAN.md acceptance criteria are covered by automated tests vs left for manual TUI verification (live footer render + real 429 can only be checked by a human running pi against a live 9router).
  - Touches: `src/index.ts`, `src/__tests__/index.test.ts` (and small helper extraction if needed for testability)

## Opus economy (call it rarely, call it complete)

All tasks in this slice are S/M and reviewed by **`sonnet`** — no L-tier tasks, so no routine Opus review. Opus is spawned ONLY for the two-failure circuit-breaker decision. When it is:

- **Front-load.** Put everything Opus needs in the spawn prompt — full diffs, acceptance criteria, the actual verification output, relevant file contents, and any prior attempts/decisions — so it answers in a single shot.
- **The reviewer self-verifies.** The `reviewer` agent re-runs commands and re-reads code to verify its own findings before reporting — it does not spawn a nested verifier.

## Per-task execution loop

For each task in the current batch:

1. **Implement.** Spawn `implementer` with `model: sonnet`. Give it: the task, its acceptance criteria, the worktree path (if any), and the instruction to work test-first and to run `bunx tsc --noEmit && bunx vitest run` (or the task-scoped subset) before reporting back. If you are resuming a task that handed over, pass the saved handover doc too (see step 2b).
2. **Read the implementer's `RESULT`:**
   - **`HANDOVER`** — the implementer hit its context budget and wrapped up cleanly. This is **not** a failure and must **not** increment `attempts`. Save its handover block to `plans/quota/001-9router-quota-plugin/handovers/<task-id>-<seq>.md` (so it survives a crash), then immediately spawn a **fresh** `implementer` (`model: sonnet`) for the same task, passing that handover doc as its starting context. Repeat until you get `PASS` or `FAIL`.
   - **`PASS` / `FAIL`** — proceed to step 3.
3. **Verify.** Confirm the implementer's claim yourself — actually run `bunx tsc --noEmit && bunx vitest run`. Self-reports are not evidence. If it fails, **re-run once** before counting it: a single flaky pass/fail shouldn't burn a real attempt. Two consistent fails = real fail.
4. **Review.** Spawn `reviewer` with `model: sonnet`, give it the diff + acceptance criteria. (No L tasks here, so no Opus review round.)
5. **Outcome.**
   - **Pass** (verification green AND reviewer `PASS`): merge the worktree into the integration branch (if parallel), mark the task `done`, commit, update RALPH_PROGRESS.md.
   - **Fail** (verification red OR reviewer `REJECT`): increment the task's `attempts`. If `attempts < 2`, loop back to step 1 with the failure feedback. If `attempts == 2`, trigger the circuit-breaker.

Handovers are about context, not correctness: a task may cycle through several implementers via handover docs and still be on its **first** attempt. Only verification-red or reviewer-`REJECT` counts toward the two-failure circuit-breaker.

## Two-failure circuit-breaker (Opus decides continue vs abort)

When a task fails twice, do not keep grinding. Spawn `reviewer` with `model: opus` as a **decision-maker** in a single complete call (see "Opus economy"), giving it: the task, both failure attempts and their errors, the acceptance criteria, and the relevant diffs. Require it to answer in exactly this shape:

```
DECISION: CONTINUE | ABORT
REASON: <one paragraph>
GUIDANCE: <if CONTINUE: concrete next approach. if ABORT: why this slice can't safely proceed.>
```

- **CONTINUE:** reset the task's attempts, record the guidance in RALPH_PROGRESS.md, and retry with it.
- **ABORT:** record the decision and reason in RALPH_PROGRESS.md, write the abort into the summary, clean up worktrees (keep `ralph/quota-plugin` for inspection), then emit the promise as the final line to exit cleanly. The loop must not hang — a recorded, reasoned abort is a valid terminal state.

## Finishing the slice

When all batches are `done`:

1. Run `bunx tsc --noEmit && bunx vitest run` once more on the integration branch; it must pass.
2. Ensure everything is committed and RALPH_PROGRESS.md is up to date.
3. **Clean up worktrees.** Merge any remaining reviewed child worktrees, then `git worktree remove` all of them. Keep only `ralph/quota-plugin` — no worktree litter. (Same cleanup on abort.)
4. **Cost note.** Append a one-line tally to RALPH_PROGRESS.md: iterations used, Opus calls made, tasks done.
5. Write the **merge-handoff summary** into RALPH_PROGRESS.md: what changed, how it was verified, any follow-ups (esp. which PLAN.md acceptance criteria need MANUAL TUI verification against a live 9router), and the **exact commands** a human runs to merge `ralph/quota-plugin` into `main` (e.g. `git checkout main && git merge --no-ff ralph/quota-plugin`). Do **not** merge to `main` yourself — that is the human approval gate.
6. **Push + offer PR (outward-facing — offer, never auto-create).** Push the feature branch: `git push -u origin "$(git branch --show-current)"` (never force, never push `main`). Host is `github.com`: if `gh auth status` is clean, surface (do not run) `gh pr create --base main --head "$(git branch --show-current)" --fill`; else print the compare URL `https://github.com/tigorlazuardi/9router-quota-footer/compare/main...ralph/quota-plugin?expand=1`. End the promise turn with `main ← ralph/quota-plugin` + the offered command. The PR is the human approval gate — do NOT merge to main yourself.
7. Emit the promise as the final line: the promise tag on its own line.
