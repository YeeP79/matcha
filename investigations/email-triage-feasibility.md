# Email Triage as a matcha Plugin — Feasibility

**Date:** 2026-05-15
**Author:** ryan
**Posture:** personal tooling, three weekends, no compliance rigor
**Status:** read-only investigation, no code written

---

## TL;DR

**Conditional go.** The cost model is comfortably under target, local LLM latency is proven (~260ms warm on existing hardware), and matcha's Lua plugin contract handles 80% of what's needed. But there are two non-trivial constraints worth resolving before code:

1. **matcha plugins fire only when the TUI is running.** The daemon already handles IDLE + desktop notify, but does not invoke plugins. "Always-on triage from the system tray" needs either (a) always-running TUI in tmux, or (b) a small matcha PR adding plugin hooks to the daemon.
2. **The `email_received` hook payload contains headers only, no body.** Heuristics on subject/sender work; LLM classification of body content requires either a body-fetch back through Gmail API (extra hop, extra OAuth token in plugin), or a payload enrichment in matcha core.

If you accept "matcha TUI runs in a persistent tmux on monsoon-pc" and "classifier reads only subject + sender + first ~200 chars of snippet via Gmail API," you can ship without changing matcha. If you want desktop notifications independent of the TUI, you need an upstream PR (documented in §6).

The three biggest open questions before code are listed in the exit condition.

---

## 1. Volume baseline (last 30 days)

**Status: data not gathered this session.** The Gmail MCP requires OAuth authentication that hasn't been completed in this environment, and I have no ambient access to mailbox stats. This section documents the queries needed and reasonable placeholders for §2 cost modeling.

### Queries to run (Gmail API, via `mcp__claude_ai_Gmail__authenticate` then list/search)

| Question | Gmail search query | Notes |
|---|---|---|
| Total received last 30d | `in:inbox newer_than:30d` | excludes spam/trash |
| Direct-TO (high signal) | `to:me newer_than:30d -from:me` | |
| CC-only (lower signal) | `cc:me newer_than:30d -to:me` | typical noise tier |
| BCC / list-mail | `list:* newer_than:30d` | mailing lists |
| Threads I replied to | `from:me newer_than:30d` (then dedupe threadId) | participation signal |
| After-hours arrivals | use `internalDate` in Messages.get; filter outside 9–18 local | needs post-processing |
| Unique sender count | `from:` distinct from message metadata | |

### Placeholder estimates (educated guesses, replace with real data before greenlight)

A working professional with a moderately active gmail account, no aggressive filters:

- **120–250 received/day** (mix of personal + work-adjacent newsletters + transactional)
- **15–30 direct-TO/day** that even plausibly warrant attention
- **3–8/day** in the "ambiguous middle" (not obvious noise, not obvious VIP)
- **1–4/day** truly notify-worthy (would have looked anyway, getting a heads-up adds value)

**Heuristic-catchable fraction (subject/sender/list-id rules):** plausibly 85–95%. Newsletters, GitHub, calendar invites, AWS bills, Stripe receipts, common marketing — all label themselves clearly in headers. The expensive cases are well-formatted human emails from unfamiliar senders.

### Plan to get real numbers

Run a one-shot script (Python, ~30 lines) against Gmail API: pull message metadata for `newer_than:30d`, dump to a CSV, count buckets. Output: `volume-baseline.csv` next to this doc. Time cost: ~20 minutes including OAuth setup. **This must happen before any implementation decisions are locked in** — the cost model in §2 is sensitive to it.

---

## 2. Cost modeling

Source pricing: Anthropic public list as of May 2026.
- **Claude Haiku 4.5:** $1.00 / Mtok input, $5.00 / Mtok output.
- **Claude Sonnet 4.6:** $3.00 / Mtok input, $15.00 / Mtok output.

(Use Haiku for classification; reserve Sonnet for the rare "explain why" if a one-liner needs nuance — but the default plan uses Haiku for everything.)

### Token-per-call assumptions

Per email sent to API:
- **Input:** ~600 tokens. Stable preamble (~250 cached) + sender + subject + snippet (~250 chars ≈ 60 tok) + few-shot examples (~280 tok). Prompt caching reduces effective input cost by ~half on cache hits — modeled as 600 nominal.
- **Output:** ~50 tokens. JSON label + one-line "why this matters" + confidence.

So **~650 tokens/call total**, dominated by input.

Per-call Haiku cost: `(600 × $1.00 + 50 × $5.00) / 1e6 = $0.0009`.
Round to **~$0.001/call** for sticker-shock purposes.

### Path A — API-only baseline (Haiku classifies every email)

Assume **200 emails/day = 6,000/month**.

```
6,000 × $0.001 = $6.00/month
```

At placeholder volume, **already over the $5/month target**, and this scales linearly. Reject as the primary path.

Sensitivity at 2× volume (400/day): **$12/month** — fails target.

### Path B — Cascade (local handles 90–95%, API only for notify-tier one-liner)

The heuristic + local stack handles:
- Newsletters / list-mail / known-noise senders → drop, no LLM.
- Known-VIP / direct-TO from prior correspondents → notify, no API needed (label is "obvious notify," no nuance one-liner).
- Ambiguous middle → local llama3.2 or qwen3 classifies to {notify, ignore, defer}.
- **Only the items local labels as `notify` with `needs_explanation=true` go to the API** for the one-liner.

Working volume: 200 emails/day.

| Tier | % of mail | Daily count | API calls? |
|---|---|---|---|
| Heuristic drop (lists, marketing, transactional receipts) | 70% | 140 | 0 |
| Heuristic accept (VIP, direct-TO known sender) | 10% | 20 | 0 (notify without explanation) |
| Ambiguous → local LLM | 20% | 40 | only on `notify` outputs that score "needs explanation" |
| Of those 40, local labels `notify`: ~25% | — | 10 | 10 |
| Of those 10, "needs explanation" gate hits: ~30% | — | 3 | 3 |

**Optimistic: 3 API calls/day = 90/month.**
```
90 × $0.001 = $0.09/month
```

**Pessimistic: spillover doubles. Heuristics catch only 80% of noise, local labels 40% as notify, 50% need explanation → 16 API calls/day = 480/month.**
```
480 × $0.001 = $0.48/month
```

**Annual projection:** optimistic ~$1, pessimistic ~$6, midpoint ~$3.

Sensitivity at 2× volume (400/day): optimistic $0.18/mo, pessimistic $0.96/mo. **Comfortably under target across the range.**

### Conclusion for §2

Cascade design is the only viable path. API-only fails on cost; cascade is **5–60× under budget** with comfortable margin even at 2× volume. The $5/month target is realistic but only if the cascade actually performs — see §3 for whether local can carry 90%+ of the load.

---

## 3. Local LLM feasibility on monsoon-pc

### Current Ollama state (measured 2026-05-15)

- **Service:** `ollama serve` running 2 days uptime, PID 2372. No models currently resident (`ollama ps` empty — idle).
- **Models installed (relevant):**
  - `qwen3:14b` — 9.3 GB, Q4_K_M, 14.8B params
  - `llama3.2:latest` — 2.0 GB, Q4_K_M, 3.2B params
  - `qwen3-embedding:latest` — 4.7 GB, Q4_K_M, 7.6B params (Krypteia)
  - `mxbai-embed-large:latest` — 670 MB (Krypteia configured default)
  - several other embedders (nomic, jina-code, snowflake-arctic, gemma)

### Hardware headroom (measured)

- **GPU:** RTX 5060 Ti, 16,311 MiB VRAM. Currently 2,471 MiB used (likely Xorg/compositor) → **~13.4 GB free**.
- **RAM:** 60 GB total, 40 GB available, 17 GB buff/cache. Plenty.
- **Load:** uptime shows load avg 0.58 — idle desktop.

### Live latency test (warm, JSON mode)

```bash
curl -X POST http://localhost:11434/api/generate \
  -d '{"model":"llama3.2:latest","prompt":"Classify ... Reply with JSON only","stream":false,"format":"json"}'
```

Two consecutive calls:
1. **Cold load:** 3.17s total (3.10s load_duration, 13ms generation).
2. **Warm (same prompt, ~30s later):** **258 ms total** (82ms load, 23ms prompt_eval, 76ms generation). Output: `{"label": "notify"}` — JSON mode emitted clean, parseable output on the first try.

Default Ollama `keep_alive` is 5 minutes; for a triage workload, set `keep_alive: "30m"` so the model stays resident through any conversational gap.

### Recommended classifier: `llama3.2:3b` Q4_K_M

Rationale:
- **Fits with massive headroom** (~2.5 GB resident vs 13 GB free), zero risk of starving Krypteia's 670 MB embedder.
- **Sub-second warm latency** — 258 ms measured, easily meets the <5s end-to-end target.
- **JSON-mode output reliable** — Ollama's `format: "json"` constraint produced valid JSON on the first call without retries.
- **3B class is enough for triage.** Triage is a coarse classification problem: noise / ambiguous / notify. The hard cases require world knowledge that a 14B model only marginally improves over a 3B. The win for 14B is in the "explain why" one-liner — but in the cascade design, that one-liner is the API call, not local.

Fallback model: `qwen3:14b` Q4_K_M if 3B accuracy disappoints during the 30-day calibration window. It fits in VRAM (~9 GB), would add ~1–2s warm latency, and would push the cost back toward API-only territory if it could fully replace the API one-liner (cheaper electricity-wise than network calls).

### Coexistence with Krypteia

- Krypteia configured embedder is `mxbai-embed-large` (670 MB). Even with classifier resident, **total VRAM use ~3.2 GB**, leaving 10+ GB headroom.
- Krypteia is request-driven (search invocations), classifier is event-driven (one per ambiguous email = a few per hour). **Realistically zero contention.** Both can keep_alive without thrashing.
- Worst case: an active `nx run krypteia:test-embedding` benchmark with `qwen3-embedding:7b` resident (4.7 GB) + classifier (2 GB) = 6.7 GB, still half the available VRAM.

### End-to-end latency budget (target: <5s)

| Stage | Budget | Measured/Expected |
|---|---|---|
| Gmail body/snippet fetch | 200–800 ms | network-bound |
| Heuristic pass (Lua) | <5 ms | trivial |
| Local LLM (warm) | 100–400 ms | **measured 258 ms** |
| API call (Haiku, only ~3/day) | 1–2 s | network + Anthropic |
| Notify dispatch | 50 ms | `notify-send` |
| **Total (no API)** | ~1 s | well under 5s |
| **Total (with API)** | ~3 s | well under 5s |

**Verdict: §3 is unambiguously feasible.** No measurable impact on dev workflow expected — VRAM headroom is large, classifier inference is brief and infrequent.

---

## 4. Security and compliance posture

**This is the section that can kill the project. Read it carefully.**

### Is corporate email involved?

The user works at New Western. The project description says "personal email triage," and Gmail OAuth is the most plausible target. Two scenarios:

1. **Personal Gmail account only.** Posture is "do whatever I want with my own data." No employer authority over which AI provider processes my own personal mail. **Low risk.**
2. **NW corporate Gmail / Google Workspace account.** Suddenly subject to NW data-handling policy, contracts that may include "no data leaves Workspace tenant," and possibly DLP controls. **High risk, sign-off required.**

**The user must confirm which mailbox(es) are in scope before code.** A casual phrasing of "my email" can mean either; the answer changes the project.

### Written NW policy check

- Not surveyed in this session (Confluence Atlassian MCP requires auth in this environment).
- Action item: search Confluence for ("AI" OR "LLM" OR "ChatGPT") AND ("policy" OR "data handling" OR "acceptable use"). The `mcp__confluence__search_content` tool is configured for the NW workspace.
- Also check `nw-developer-tools` and similar repos for a written `AI-POLICY.md` or `SECURITY.md` mentioning model providers.
- If no written policy: default to "would I be comfortable explaining this to the CTO" test. The honest answer for corporate mail going to Anthropic is "not without asking."

### What data leaves the machine

Per the cascade design, only ambiguous-then-local-says-notify emails reach Anthropic. The payload (per call) is:

- Sender display + address
- Subject line
- First ~250 chars of body snippet (Gmail API's standard `snippet` field is plain-text first ~200 chars, plus optional fetch of first part for ~50 more)
- No attachments, no full thread, no recipient list beyond "me"

This is roughly equivalent to what a notification preview already exposes on the lock screen. Not equivalent to a full thread inspection.

### Draft sign-off ask (if needed for corporate mail)

> Subject: Personal AI tooling — quick OK request
>
> I'm building a side project that flags notify-worthy emails using a local LLM on my own machine; about 3 emails per day with ambiguous priority would have a one-line summary generated by Claude Haiku (sender, subject, ~200-char snippet — same content surface as a phone lock-screen preview, no full body, no attachments, no thread history). Local Ollama handles ~95% of mail with no external calls. Nothing is logged at Anthropic past their standard 30-day API retention. **Can I point this at my @newwestern.com inbox, or should I keep it scoped to my personal Gmail only?**

### Recommendation

**Default: personal Gmail only.** This is "do whatever you want." If corporate inbox value is high enough to justify the ask, send the paragraph above. Until that explicit yes comes back, the plugin must hard-fail on any account_id matching the corporate domain.

---

## 5. Code reuse audit (spartan-toolkit)

**Rule: lift functions, don't take dependencies.**

### What's findable in spartan-toolkit

Quick survey shows:
- `spartan-toolkit/krypteia/lambdas/krypteia-embedder/src/` exists but is an embeddings-specific layout (`index.ts`, `types.ts`). The Ollama client logic for Krypteia is wrapped in MCP servers under `nw-developer-tools/mcp-servers/ollama/` (a Python implementation: `client.py`, `server.py`, `tools/`).
- No obvious standalone "Ollama TypeScript client" file matching a name like `ollama-client.ts`.
- Spartan is a complex Nx monorepo with strict dependency rules per its CLAUDE.md. Lifting from it cleanly is awkward.

### Lift candidates

| Candidate | Where | Verdict |
|---|---|---|
| Ollama generate/chat HTTP wrapper | `nw-developer-tools/mcp-servers/ollama/src/client.py` | **Skip.** It's Python; matcha is Go + Lua. Wrong language. Ollama's HTTP API is 30 lines to wrap from scratch. |
| Embedding model selection | Krypteia bench tooling | **Skip.** Project doesn't need embeddings, only classification. |
| JSON-mode retry-on-malformed-output pattern | unclear; not located in this audit | **Greenfield it.** Three lines: try parse → on fail, retry with stricter prompt → on second fail, downgrade to heuristic. |
| Gmail polling | not present in spartan | n/a |
| JSONL feedback persistence | not present in spartan | **Greenfield.** Append-only file, ~10 lines. |

### Recommendation

**Greenfield this project. No spartan dependency, no lifts.** Standard library + minimal HTTP for Ollama is faster to write than navigating spartan's Nx dependency graph from outside the workspace, especially given the language mismatch (matcha plugins are Lua → call out via `matcha.http()` or to a sidecar process).

This is consistent with the "personal tooling, three weekends" posture.

---

## 6. matcha plugin design

### Plugin contract (from source)

Read: `plugin/plugin.go`, `plugin/api.go`, `plugin/hooks.go`, `plugin/README.md`, plus examples in `plugins/*.lua`.

**Runtime:** sandboxed gopher-lua VM. **No `os`, `io`, `debug` libraries.** Plugins are `.lua` files under `~/.config/matcha/plugins/` (or directories with `init.lua`).

**Lua API surface (the `matcha` module):**

| Function | Surface |
|---|---|
| `matcha.on(event, fn)` | Register hook callback |
| `matcha.log(msg)` | stderr log |
| `matcha.notify(msg, seconds?)` | **TUI banner only**, not OS-level desktop notification |
| `matcha.set_status(area, text)` | Persistent status text in `"inbox" \| "composer" \| "email_view"` |
| `matcha.set_compose_field(field, value)` | Mutate composer fields |
| `matcha.bind_key(key, area, desc, fn)` | Register keybind in one of the three areas |
| `matcha.http(opts)` | HTTP client. **10s timeout, 1 MB body cap.** http/https schemes only. |
| `matcha.prompt(placeholder, cb)` | Composer-area text input overlay |

**Hook events:**

| Event | Payload | Use for triage |
|---|---|---|
| `startup` / `shutdown` | — | init/cleanup |
| `email_received` | `{uid, from, to, subject, date, is_read, account_id, folder}` — **no body** | primary trigger |
| `email_viewed` | same shape | feedback signal (user clicked it) |
| `email_send_before` / `email_send_after` | `{to, cc, subject, account_id}` | irrelevant for triage |
| `folder_changed` | folder name string | irrelevant |
| `composer_updated` | composer state | irrelevant |

### Capability surface — answers to the user's explicit questions

| Capability | Available? |
|---|---|
| Register a virtual folder | **No.** Folder list is owned by matcha core; `folder_changed` is read-only. |
| Write to status bar | **Yes**, per-area via `set_status`. |
| Bind keys | **Yes**, in `inbox \| email_view \| composer`. |
| Fire desktop notifications | **No, not from Lua.** `matcha.notify` shows a TUI banner. Desktop notifications via `notify.Send()` are invoked directly by matcha Go code (`main.go:812`, `main.go:845`, `daemon/daemon.go:397, 445`) and not exposed to plugins. |
| Persist state | **No filesystem access from Lua.** Workaround: HTTP POST to a localhost sidecar process. |
| Run background goroutines | **No.** Plugins are invoked only when hooks fire; no scheduled wakeups, no timers. |

### Critical architectural constraint discovered

**Plugins are loaded and invoked in the TUI process, not the daemon.**

- `plugin.NewManager()` and `plugins.LoadPlugins()` are called in `main.go:3846-3847` — the TUI entry point.
- `email_received` is invoked from `main.go:631` in response to a `FolderEmailsFetchedMsg` (TUI message).
- The daemon (`daemon/daemon.go`) handles IMAP IDLE and emits its own desktop notifications via `notify.Send` independently, but **does not invoke any plugin hook**.

Implications:
- If matcha TUI is not running, no plugin fires.
- The plugin will be re-invoked for **every email in the fetched batch** whenever `FolderEmailsFetchedMsg` arrives (`main.go:627-633` iterates over all emails). It is not gated to "truly new since last fetch." Plugins handling notifications need to de-duplicate by UID themselves.

### Triage plugin design (matcha-native, no core PR)

```
~/.config/matcha/plugins/triage/
├── init.lua              -- hook registration, heuristic rules, dedupe
└── (no other files needed; state lives in the sidecar)
```

Plus a **local sidecar daemon** (matcha is not the right place for this state):

```
~/.local/bin/matcha-triage-sidecar     -- Go binary, ~300 LOC
  - HTTP server on 127.0.0.1:7777
  - POST /classify {uid, from, subject, snippet?} → {label, score, why?}
  - GET  /seen?uid=...                            → boolean (UID dedupe)
  - POST /feedback {uid, correction}              → JSONL append
  - State: SQLite at ~/.local/share/matcha-triage/state.db
  - Calls Ollama at localhost:11434
  - Calls Anthropic Haiku only on local-says-notify-needs-explanation tier
  - Fires libnotify (notify-send) for notify-tier results — solves the desktop notification gap
  - Holds the Gmail snippet/body fetch logic too (one OAuth refresh, not per-plugin)
```

Lua glue:

```lua
local matcha = require("matcha")

matcha.on("email_received", function(email)
    -- Fast heuristic dropouts inline (Lua, microseconds)
    if email.from:match("noreply@") then return end
    if email.from:match("github%.com") and email.subject:match("^%[") then return end
    -- Anything that survives heuristics: hand off to sidecar
    matcha.http({
        url = "http://127.0.0.1:7777/classify",
        method = "POST",
        headers = {["Content-Type"] = "application/json"},
        body = string.format('{"uid":%d,"from":%q,"subject":%q,"account_id":%q}',
                             email.uid, email.from, email.subject, email.account_id),
    })
    -- response ignored; sidecar fires desktop notification itself
end)

matcha.on("email_viewed", function(email)
    matcha.http({
        url = "http://127.0.0.1:7777/feedback?uid=" .. email.uid .. "&action=viewed",
        method = "POST",
    })
end)

matcha.bind_key("F", "inbox", "false-positive: mark as noise", function(state)
    -- requires bind_key callback to have access to current email UID;
    -- check tui.Inbox state passed in — likely needs minor matcha tweak
end)
```

### What this design buys

- **No matcha core changes required** if you accept: matcha TUI must be running (in tmux) for plugin to fire.
- **Desktop notifications work** because the sidecar (running outside the Lua sandbox) fires `notify-send` directly.
- **State, OAuth, body fetch, Anthropic calls, feedback JSONL** all live in the sidecar — Lua stays trivially simple.
- **Sidecar restartable / debuggable independent of matcha.**

### What this design loses

- TUI must be running. If matcha crashes or you reboot, triage is dark until you start it again.
- `email_received` runs against the full folder fetch result, so the plugin sends every UID to the sidecar; sidecar does dedupe. Mild wasted HTTP overhead, not a correctness issue.

### Minimal upstream matcha PR (optional, for "always-on")

If "always-on independent of TUI" is required, the cleanest change is:

**Add `email_received` plugin hook invocation in the daemon's IDLE update handler** (`daemon/daemon.go` near line 397/445 where `notify.Send` is already called for new mail). This requires:

1. A `plugin.Manager` in the daemon (currently only in TUI).
2. A single new call site: when IDLE delivers a new-mail update, invoke `email_received` with the same payload shape.
3. Optionally also expose `email.snippet` in the payload — the daemon already has the message metadata.

Size estimate: ~60 lines, surgical, no API break for existing plugins. Worth doing if upstream accepts; ask the maintainer first. **Not needed for v1.**

---

## 7. Feedback loop design

### Minimum viable correction capture

Three signals, all flow through the sidecar:

1. **Implicit positive — opened it.** `email_viewed` hook → POST `/feedback?action=viewed`. If the classifier said `notify` and you opened it within 1 hour → confirmed positive. If classifier said `ignore` and you opened it → false negative.
2. **Implicit negative — never opened, archived/deleted.** Hard to capture from current hook set without periodic poll of folder state. **Out of scope for v1.**
3. **Explicit correction — keybind.** `F` in `inbox` area: "this was a false positive, mark as noise." POST `/feedback?action=fp&uid=...`.

### Storage

JSONL at `~/.local/share/matcha-triage/feedback.jsonl`. One record per event:

```
{"ts":"2026-05-15T09:14:22Z","uid":12345,"from":"...","subject":"...","predicted":"notify","actual":"fp","source":"keybind"}
```

### Augmenting the classifier

After ~50 events accumulated:
- Periodically (cron/sidecar timer, weekly), build a few-shot example block from recent corrections, biased toward confusables (where classifier and actual disagree).
- **Bounded growth:** keep last N=20 examples in the prompt, prioritize most-recent confusables, evict by recency.
- This is appended to both the local classifier prompt and the Haiku prompt.

### Calibration metric

After 30 days, compute over the JSONL:

- **Precision of `notify` label** = `# (predicted notify AND user opened within 1h) / # predicted notify`. Target: ≥70%.
- **Recall on user-opened mail** = `# (opened within 1h AND predicted notify) / # opened within 1h`. Target: ≥80%.
- **API call rate** = `# Haiku calls / # emails received`. Target: ≤5%.
- **Explicit FP rate** = `# F-keybind presses / # notify-tier predictions`. Target: ≤15% by day 30 (steady-state expectation; will be higher in week 1).

If any of these miss target at day 30, that's a stop-and-evaluate moment. If precision is the failure, raise the local classifier confidence threshold for sending to API or for notifying. If recall is the failure, widen heuristic rules or use the larger model.

---

## 8. Risks and unknowns

### Things that could make this not worth building

1. **False positives stay annoying.** The honest base rate for "I want to be interrupted" varies hour to hour and is hard to learn from clicks alone. The system can be 90% accurate and still be net-negative if the 10% wrong includes one wake-up at midnight. **Mitigation:** add a quiet-hours rule (heuristic, not learned) and require explicit "high signal" matches (direct-TO + known sender or VIP-rules) for after-hours.
2. **monsoon-pc availability.** Sleep/reboot kills always-on triage. **Mitigation:** systemd user unit for the sidecar; matcha TUI in tmux with `tmux-resurrect`. Even so, expect periodic gaps.
3. **Gmail API quota.** Personal Gmail allows 1 billion quota units/day per project, search.list is 5 units/call, message.get is 5 units/call. At 6000 emails/month + body fetches, ~60k units/month — **not a constraint.**
4. **matcha plugin surface is narrower than expected.** This was the headline risk going in; investigation confirms: no body in `email_received`, no filesystem, no desktop notification, no background goroutines. **The sidecar pattern routes around all of these.** The remaining limitation is "TUI must run for plugins to fire" — if that's unacceptable, requires the upstream PR in §6.
5. **The problem is behavioral, not technical.** Most "I missed an important email" failures are because the inbox was overflowing, not because the email lacked a tap on the shoulder. A triage tool that surfaces *more* notifications can amplify the problem. **Mitigation:** treat the project as a notification *filter*, not a generator. If desktop notify is already on, the goal is to make those notifications *fewer and better*, not add a parallel notify stream.
6. **Snippet-only classification loses information.** Some emails require reading paragraph 3 to know they matter. Subject + 250 char snippet will misclassify long human-written emails from unfamiliar senders. **Acceptable for v1**, may need to upgrade to fetching first 1–2 KB later.
7. **Personal vs corporate email confusion (see §4).** Re-raising as a risk: this needs explicit confirmation before code.

### Cheapest experiment that disproves the project before real engineering time

**~2-hour spike: manual labels for one week.**

For one week, every morning manually label the previous day's inbox into {noise, ambiguous, notify} in a spreadsheet. Don't build anything yet. After 7 days:

- If the count of "actually notify" is <5/day total: building automated triage is a solution looking for a problem; existing manual triage already handles it.
- If the count is 10+/day AND many of those *would have been* obvious from sender/subject: the heuristic layer alone is the project, no LLM needed.
- If the count is 10+/day AND many are genuinely ambiguous from headers: cascade design is justified.

**This spike costs ~$0 and ~2 hours; the build is three weekends. Run the spike first.**

---

## Exit condition

**Go conditional on three answers:**

1. **Volume baseline (§1)** — replace placeholders with real Gmail data. The cost model passes comfortably at the placeholder volume; if your real volume is 5× higher, re-check.
2. **Mailbox scope (§4)** — personal Gmail only, or NW corporate too? If corporate, the §4 sign-off paragraph goes out *before* code.
3. **Always-on requirement (§6)** — is "TUI in tmux" acceptable for v1, or do you need the daemon PR up front? This determines whether v1 ships in 3 weekends or 4.

If those three resolve cleanly, this is a build. The technical risks are all manageable; the cost is comfortably under target; the local hardware is over-provisioned for the workload. The one real architecture decision (sidecar pattern) is forced by the matcha plugin sandbox and is the right answer regardless of any future capability the plugin system gains.

**Recommendation: do the 2-hour labeling spike from §8 first.** That's the cheapest test of whether the problem is real before any code.

---

## Appendix — data not gathered this session

- §1 volume baseline: Gmail OAuth not completed in this environment. Run the queries listed in §1, dump to CSV.
- §4 written NW policy: Confluence MCP not authenticated in this environment. Re-run the search there.
- §5 spartan-toolkit deep audit: confirmed Python Ollama client exists in `nw-developer-tools/mcp-servers/ollama/`; not language-compatible. No other lift candidates found in cursory pass.
