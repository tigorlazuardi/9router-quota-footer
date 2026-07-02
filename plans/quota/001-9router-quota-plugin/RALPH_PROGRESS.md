# Ralph State — quota/001-9router-quota-plugin

- **status:** `planning`
- **orchestrator_model:** `sonnet`
- **base-branch:** `main`
- **integration-branch:** `ralph/quota-plugin`
- **verify-command:** `bunx tsc --noEmit && bunx vitest run`
- **install-cmd:** `bun install`
- **parallel-cap:** `4`
- **last-updated:** `2026-07-02T00:00:00Z` (iteration `0`)
- **last-progress-iteration:** `0` (no-progress guard: +3 → circuit-breaker)

> NOTE: baseline truth signal is NOT green until task `b1-bootstrap` lands package.json + tsconfig + vitest + smoke test. B1 is the bootstrap that MAKES the signal green; every later batch must keep it green.

## Task ledger

| task-id | batch | tier | reviewer | worktree | status | attempts | notes |
|---------|-------|------|----------|----------|--------|----------|-------|
| b1-bootstrap  | B1 | S | sonnet | — (on integration branch) | todo | 0 | establishes truth signal |
| b2-config     | B2 | S | sonnet | yes | todo | 0 | baseUrl resolver + debug logger |
| b2-types      | B2 | S | sonnet | yes | todo | 0 | QuotaAdapter + QuotaResponse types |
| b3-cc-adapter | B3 | M | sonnet | yes | todo | 0 | footer format + countdown |
| b3-wakeup     | B3 | M | sonnet | yes | todo | 0 | retry-after parse + timer scheduler |
| b4-index      | B4 | M | sonnet | — (on integration branch) | todo | 0 | wiring: footer + wakeup hooks |

`status` ∈ `todo | doing | done | failed`.

Batch order: B1 → B2 (2 parallel) → B3 (2 parallel) → B4. No batch exceeds cap 4.

## Progress log (heartbeat — one line per iteration)

- `iteration 0` · contract authored by Opus planner; loop not yet started.

## Decision log (circuit-breaker)

<!-- Append one entry each time the Opus circuit-breaker is invoked. -->

## Merge-handoff summary (filled at the end)

- **Outcome:** `<completed | aborted: reason>`
- **Cost tally:** `<iterations used> iters · <opus calls> opus calls · <n>/<total> tasks done`
- **What changed:** <bullet summary>
- **How verified:** <command + result>
- **Follow-ups / known gaps:** which PLAN.md acceptance criteria need MANUAL TUI verification against a live 9router (live footer render, real 429 wakeup end-to-end).
- **To merge (human runs this):**
  ```
  git checkout main
  git merge --no-ff ralph/quota-plugin
  ```
