# Plan — 9router Quota Footer Plugin

Pi extension (new package, standalone) that shows 9router subscription quota in the footer
while the active model is a 9router model. Adapter pattern: an adapter OWNS the quota
display-text generation per plan/provider; the plugin owns fetching, provider/model detection,
throttling, and footer rendering. Claude Code adapter only for now; pattern stays open for Codex etc.

## Goals / non-goals

- **Goal:** compact footer line `cc 52% · session 1h41m · weekly 20% · 2d14h` while a
  `9router/cc/...` model is active. (`52%` = session used-%, `1h41m` = session reset, `20%` =
  weekly used-%, `2d14h` = weekly reset.) `weekly sonnet` is intentionally dropped (never depletes).
- **Goal:** adapter pattern — adapter builds the SENTENCE, plugin only displays it.
- **Goal:** footer HIDDEN unless the active model's provider is `9router`.
- **Non-goal (now):** Codex adapter, other providers — leave the registry open, don't implement.
- **Non-goal:** auth — loopback single-user, security OFF on 9router. No tokens/headers.
- **Non-goal:** touching the vendor `pi-9router-ext` plugin. This is a separate package.

## Confirmed facts (from research)

### 9router API (security off, base `http://localhost:20128`)
- `GET /api/providers/client` → `{ connections: [{ id, provider, name, isActive, priority, expiresAt }] }`.
  - For Claude Code, `provider === "claude"`.
- `GET /api/usage/{connectionId}` → quota JSON:
  ```json
  {
    "plan": "Claude Code",
    "quotas": {
      "session (5h)":      { "used":52, "total":100, "remaining":48, "remainingPercentage":48, "resetAt":"<ISO>", "unlimited":false },
      "weekly (7d)":       { "used":20, "total":100, "remaining":80, "remainingPercentage":80, "resetAt":"<ISO>" },
      "weekly sonnet (7d)":{ "used":0,  "total":100, "remaining":100,"remainingPercentage":100,"resetAt":"<ISO>" }
    },
    "extraUsage": { ... }
  }
  ```
- Base URL: reuse the same source the vendor ext uses — env `NINE_ROUTER_BASE_URL` else
  `~/.pi/agent/9router-config.json` `baseUrl` else `http://localhost:20128`. Read it; do NOT
  import the vendor package.

### Model → adapter detection (the key design)
- Active model id is prefixed: Claude Code = `cc/...` (confirmed via `GET /v1/models`),
  Codex = `cx/...` (future).
- Pi exposes the active model as `event.model.provider` / `event.model.id`.
  - provider for 9router models = `"9router"` (the vendor ext registers `pi.registerProvider("9router", …)`).
  - id = e.g. `cc/claude-opus-4-8`.
- **Footer visible iff `event.model.provider === "9router"`.**
- **Adapter chosen by id prefix:** `cc/` → ClaudeCodeAdapter. Unknown prefix → no adapter → render nothing.
- **Prefix → connection.provider map:** `cc/` → `claude`. Pick the matching active connection from
  `/api/providers/client`, take its `id`, then `GET /api/usage/{id}`.

### Pi extension API (from docs/extensions.md)
- Footer segment: `ctx.ui.setStatus("9router-quota", text)`; clear/hide with
  `ctx.ui.setStatus("9router-quota", undefined)`. (NOT `setFooter`, which replaces the whole footer.)
- Read active model at startup: `ctx.model` (in `session_start`). React to changes: `model_select`
  event (`event.model`, `event.source` = "set" | "cycle" | "restore").
- Other refresh triggers available: `session_start`, `turn_end` / `before_provider_request`.
- `ctx.signal` for abort-aware fetch during a turn.

## Architecture

```
plugin (orchestration)                         adapter (text)
──────────────────────                         ─────────────
detect active model (session_start/model_select)
  └─ provider == "9router"? ── no ─► setStatus(undefined)  // hide, done
        │ yes
  pick adapter by id prefix (cc/ -> claude)
  resolve connection id (provider client) [cached]
  fetch /api/usage/{id}  [THROTTLED]
        └─ raw quota JSON ──────────────────►  adapter.format(quota) -> string
  setStatus("9router-quota", string)  ◄────────┘
```

### Adapter contract (open for future providers)

```
interface QuotaAdapter {
  // which 9router model-id prefix this adapter handles, e.g. "cc/"
  prefix: string;
  // 9router connection.provider this maps to, e.g. "claude"
  connectionProvider: string;
  // build the footer sentence from raw /api/usage JSON; return undefined to render nothing
  format(quota: QuotaResponse): string | undefined;
}
```

- Registry: `const adapters: QuotaAdapter[] = [claudeCodeAdapter]`. Lookup by
  `adapters.find(a => modelId.startsWith(a.prefix))`.
- **ClaudeCodeAdapter** (only one implemented):
  - `prefix: "cc/"`, `connectionProvider: "claude"`.
  - `format`: read ONLY `quotas["session (5h)"]` and `quotas["weekly (7d)"]`. Ignore
    `weekly sonnet (7d)` entirely (never depletes). Build EXACTLY:
    `cc {sessionUsedPct}% · session {sessionCountdown} · weekly {weeklyUsedPct}% · {weeklyCountdown}`
    e.g. `cc 52% · session 1h41m · weekly 20% · 2d14h`.
  - **Used-%** = `used` (total is 100 so `used` already equals the used-percent); be robust if
    total ever differs: `usedPct = round(100 * used / total)`. This is USED going 0→100%, not remaining.
  - Countdown computed locally from `resetAt` (ISO) vs now → `1h41m` / `2d14h`. No extra fetch.
  - If a window is missing/`unlimited`, degrade gracefully (omit that segment).
  - Owns its wording entirely — Codex's future adapter will phrase its own quotas differently.

### Refresh strategy — event-driven + throttle (per user)
- Triggers that REQUEST a refresh: `session_start`, `model_select` (when new model is `9router/...`),
  and turn boundary (`turn_end` or `before_provider_request`).
- **Throttle:** at-most-once per N seconds (default ~10–15s). A trigger inside the window reuses the
  last fetched quota; only the countdown text is recomputed locally each render.
- No fixed polling timer. Idle sessions don't fetch.
- Connection-id lookup (`/api/providers/client`) cached for the session; re-resolve only if the
  cached id 404s on `/api/usage`.

### Hide/show rules
- provider != `9router` → `setStatus(undefined)` (hidden). Switching away from a 9router model hides it.
- provider == `9router` but no adapter for the id prefix → hidden (nothing to render).
- fetch error / network down → hidden or a quiet `cc quota —` (decide in impl; must be offline-safe,
  never throw into the footer).

## Telemetry (part of done — see telemetry-planning skill)

Lightweight; this is a local UI plugin, but still instrument the moving parts:
- **Logs** (debug-level, gated by an env like `NINE_ROUTER_QUOTA_DEBUG=1`): each fetch (url, status,
  latency_ms), throttle-skip, adapter-selected (prefix), provider-hide. Redact nothing sensitive here
  (no secrets; connection id is an opaque local id, tier B — keep visible for debugging).
- **No metrics/tracing backend** in a TUI plugin by default; if the user later wants OTel, emit a
  span per fetch (`ninerouter.quota.fetch`, attrs: prefix, status, latency). Offer, don't assume.
- Failure visibility: a failed fetch must surface in debug log with the exact error + status, and the
  footer must show a non-misleading state (hidden or `—`), never a stale "fresh" number.

## Files (proposed, standalone package)

```
pi-9router-quota/
  package.json            # name, pi.extensions entry, deps (none beyond pi runtime ideally)
  src/index.ts            # plugin: events, throttle, fetch, footer render, hide/show; wires wakeup hook
  src/adapters/types.ts   # QuotaAdapter, QuotaResponse types
  src/adapters/claude-code.ts  # ClaudeCodeAdapter (the only one now)
  src/config.ts           # resolve baseUrl (env > 9router-config.json > default)
  src/wakeup.ts           # rate-limit wakeup: 429 hook, timer, sendUserMessage resume (Feature 2)
  README.md
```

Install via `settings.json` `packages` (git or local), consistent with the user's other plugins.
Track in chezmoi after it works.

## Acceptance criteria

1. With a `9router/cc/...` model active, footer shows EXACTLY `cc {s}% · session {sReset} · weekly {w}% · {wReset}`
   with live local countdowns; `weekly sonnet` is NOT shown.
2. Switch model to a non-9router provider → footer segment disappears.
3. Switch to a `9router/cx/...` (no adapter) → footer hidden (no crash).
4. Fetch is throttled: rapid triggers within the window do not spam `/api/usage`.
5. 9router down / offline → no throw, footer hidden or `—`, debug log records the error.
6. Vendor `pi-9router-ext` untouched; no import of it.
7. Adapter owns the sentence; plugin contains zero Claude-Code-specific wording.

## Open question for impl (low risk)

- Exact shape of `ctx.model` at `session_start` (provider/id accessors) — doc is terse; the implementer
  should introspect the live type during build and read provider+id the same way `model_select` exposes them.

---

# Feature 2 — Rate-Limit Wakeup

Same package, separate concern from the footer. When THIS session hits a provider rate limit, the
plugin schedules a wakeup so the SAME agent auto-resumes the previous task once the limit resets.
No `/loop`, no `/ralph`, no spawned process — pure event hook + timer + in-session message injection.

## Goals / non-goals

- **Goal:** on a `429`, compute time-to-reset, and when it elapses inject a wakeup message into the
  same session that triggers a turn so the agent continues where it left off.
- **Goal:** work in a plain interactive session — NOT tied to loop/ralph concepts.
- **Goal:** generic wakeup message; the agent reads its own history to recall the task (minimal state).
- **Non-goal:** disk persistence / cross-process revival. If the TUI process closes, the pending
  wakeup is lost (see Risks). Acceptable per user — this is "same agent resumes", not a durable queue.
- **Non-goal:** spawning a new agent or scheduler loop.

## Resolved decisions

- **Reset-time source:** BOTH. `429` `retry-after` header is PRIMARY; nearest 9router quota `resetAt`
  (session 5h / weekly 7d) is FALLBACK when the header is missing.
- **Who resumes:** the SAME session/agent (context still in the window).
- **Wakeup message:** GENERIC — `"rate limit reset, lanjutkan task sebelumnya"`.

## Confirmed pi API (verified in installed pi docs/examples)

- **Trigger hook:** `after_provider_response` exposes `event.status` and `event.headers`. On
  `event.status === 429`, read `event.headers["retry-after"]` (extensions.md ~L646-700). Header
  availability depends on provider/transport — hence the `resetAt` fallback.
- **Wakeup:** `pi.sendUserMessage(text)` (always triggers a turn when idle) or
  `pi.sendMessage({...}, { triggerTurn: true })` injects into the SAME session and runs a turn
  (extensions.md ~L1353-1400).
- **Precedent:** `examples/extensions/file-trigger.ts` — watcher → `sendMessage` + `triggerTurn`.
  Wakeup is the same pattern, timer-driven instead of file-driven.
- **Timer placement:** timers are allowed INSIDE event handlers (like `fs.watch` inside
  `session_start`), but NOT in the extension factory (docs L221). Register the hook, keep the timer
  in module/handler scope.

## Architecture

```
after_provider_response(event)
  └─ event.status === 429 ?
        │ yes
  delay = parse(event.headers["retry-after"])        // seconds OR HTTP-date
        └─ missing? → delay = nearest quota resetAt − now   // fallback (reuse footer fetch)
  clear any existing timer                            // single active timer
  timer = setTimeout(delay + small jitter, () => {
     if (session still active && user hasn't resumed manually)
        pi.sendUserMessage("rate limit reset, lanjutkan task sebelumnya")  // triggers turn
  })
```

- **Single timer:** at most one pending wakeup. A new `429` clears + replaces it.
- **Cancel on manual resume:** if the user sends a prompt / the agent runs again before the timer
  fires, cancel the pending timer to avoid a double-trigger (watch `turn_start` /
  `before_agent_start` to detect activity).
- **`retry-after` parsing:** handle both forms — integer seconds and HTTP-date — per RFC 7231.
- **Sanity clamp:** ignore absurd/negative delays; optionally cap very long delays and log.

## Telemetry (part of done)

Debug-gated (same `NINE_ROUTER_QUOTA_DEBUG=1` flag):
- `429` caught (status, raw `retry-after` value).
- delay computed + SOURCE (`retry-after` vs `resetAt` fallback).
- timer scheduled / replaced / cancelled.
- wakeup fired (and whether it was skipped due to inactivity/manual-resume).

## Risks

- **Long delays (process death):** `setTimeout` dies if the TUI process closes. Fine for short
  `retry-after` (secs–mins). For a weekly (7d) reset the wakeup only fires if the session stays open.
  Inherent limit of "same agent resumes" without disk persistence — accepted (out of scope now).
- **Header availability:** some providers/transports don't expose `retry-after`; `resetAt` fallback
  covers that, but if BOTH are unavailable the wakeup can't be scheduled (log + skip).
- **Race — manual resume:** user resumes before the timer fires → must cancel the pending timer.
- **`retry-after` format ambiguity:** seconds vs HTTP-date — implementer must handle both.

## Acceptance criteria (wakeup)

1. A `429` with a `retry-after: <seconds>` header schedules exactly one timer; when it elapses the
   session receives a user message `"rate limit reset, lanjutkan task sebelumnya"` and a turn runs.
2. A `429` WITHOUT a usable `retry-after` header falls back to the nearest quota `resetAt` for the
   delay; debug log records the fallback source.
3. A second `429` before the first fires REPLACES the timer (never two concurrent wakeups).
4. If the user manually resumes (new prompt / agent activity) before the timer fires, the pending
   timer is cancelled — no double-trigger.
5. Negative/absurd `retry-after` values are clamped/ignored, not scheduled blindly.
6. All wakeup logging is debug-gated; nothing prints in normal operation.
7. Feature is independent of `/loop` and `/ralph` — works in a plain interactive session.
```
