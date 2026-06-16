# Next Workstreams (post v0.1.1)

Four scoped workstreams to take Mitable from "demonstrably works for one customer manually" to "the auto-ingestion claims in the docs are actually backed by code." Each can be done independently; ordering recommendation at the bottom.

---

## A. Real Slack adapter

**Goal:** Replace `StubSlackClient` with an adapter that proxies to a real Slack MCP server so the existing two-phase scan path (`src/ingest/slack.ts`) produces real extractions instead of always returning "no new messages."

**Why now:** The single biggest gap between v0.1.1 claims and reality. The whole `/sources` → "add channel" UX leads users to expect ingestion that doesn't actually run.

**Scope**

- New file: `src/ingest/slack-real.ts` implementing `SlackClient` (interface already defined in `src/ingest/slack-adapter.ts`).
- Discovery: detect at startup whether a Slack MCP is connected and which tool names it exposes. Two realistic paths:
  - Hard-code support for the official Anthropic Slack MCP (most common case)
  - Or: read tool names from the MCP's `tools/list` and map by pattern (`*read_channel*`, `*read_thread*`, `*search*`)
- Auth preflight per `docs/07 §1.5`: cheap call up front, fall through to `auth_expired` error if missing/expired. Surface the one-line fix in command-center status header.
- Rate-limit handling per `docs/07 §1.6`: respect `Retry-After`, retry once, skip channel on second failure, log and continue.
- Wire into `src/mcp/server.ts` `sweep_now` and `src/ingest/scheduler.ts` — replace `new StubSlackClient()` with the real one (behind a `MITABLE_SLACK_ADAPTER=real` flag so the stub stays available for tests).
- Enable the scheduler by default once the real adapter is wired (lift `MITABLE_SCHEDULER=1` gate).

**Tests**

- New eval file: `eval/carver/slack-adapter.test.ts` running the existing scan path against a recorded-fixture client (already have `CannedSlackClient` for this — confirm it still validates the scan logic with realistic message shapes).
- Manually verify with one real channel before declaring done: probe → read → classify → confirm an event appears in the event log with `source_type='slack'` and a real `evidence_text`.

**Honest unknowns**

- Slack MCP tool names. The plan called this out in milestone 6 — we still don't know the exact names. First step of this workstream is `tools/list` on the running Slack MCP and writing the adapter against what's actually there.
- Whether the MCP supports pagination cleanly. `readChannel` with `limit=200` might need cursor handling.
- Authentication scope. Reading channels often needs `channels:history` + `groups:history` + `im:history` — verify the MCP grants these.

**Out of scope**

- Stage 2 dedup (separate workstream)
- Slack DM handling (channels only, per `docs/07 §1.1`)
- Multi-workspace support

---

## B. Real Granola adapter

**Goal:** Same as A, against `https://mcp.granola.ai/mcp`. Mirror of the Slack stack.

**Why now:** For an FDE who lives in meetings more than channels, Granola is the higher-value source. The Granola adapter is also smaller — the scan is one fetch per meeting rather than channel + thread.

**Scope**

- Add Granola MCP via `claude mcp add granola --transport http https://mcp.granola.ai/mcp` (user-side, document in README).
- New file: `src/ingest/granola-real.ts` implementing `GranolaClient` (interface in `src/ingest/granola-adapter.ts`).
- Auth preflight + rate-limit handling as in A.
- Customer mapping heuristic: when a meeting hasn't been explicitly mapped to a customer, fall back to attendee-domain matching (e.g. `*@carver.com` → customer "carver" if registered).
- Wire into `sweep_now` + scheduler the same way.

**Tests**

- `eval/carver/granola-adapter.test.ts` against `CannedGranolaClient` with realistic note shapes.
- Manual verification: one real meeting in Granola → run sweep → confirm extractions land with `source_type='granola'`.

**Honest unknowns**

- Exact Granola MCP tool names. Document them once known.
- Whether Granola exposes a stable `updated_at` for watermarks or requires polling differently.
- Permissions model. If Granola requires re-auth per session, the preflight path needs to handle that gracefully.

**Order vs A**

A and B share design. Realistic approach: build A first to settle the adapter pattern, then B is a copy with one search-and-replace. Doing them as one workstream is reasonable; doing them as two is more digestible.

---

## D. Session-end auto-customer mapping

**Goal:** Make the existing SessionEnd hook + classification queue actually populate profiles automatically, without requiring the user to call `drain_classifications` manually with a `customer_id`.

**Why now:** This is the only background source that **already works without an external MCP** — the hook is wired, the queue is real, the classifier shells out to `claude -p`. The single missing piece is "which customer was this session about?"

**Scope**

The hook already captures `cwd` and queues sessions. What's needed:

1. **Customer inference order** (try each, stop at the first match):
   - Did the user invoke `/mitable <customer>` in this session? If yes → that customer. (Requires recording `/mitable` invocations to a small `~/.mitable/recent-customer.json` keyed by session_id.)
   - Does the session's CWD match a registered customer's `cwd_pattern`? (New: customers can have a list of cwd patterns; `add_customer` MCP tool takes an optional `cwd_patterns` array.)
   - Did the transcript mention a known customer name >= 3 times? (LLM-based; runs as part of the classifier pass.)
   - Otherwise: leave on queue with `customer_id_hint=null`, surface in command-center queue view for manual assignment.

2. **Scheduler auto-drain.** The `src/ingest/scheduler.ts` 5-min loop should call `drain_classifications` automatically for queue rows that resolved to a customer. Rows that didn't resolve stay pending.

3. **Command center UI:** `/queue` page (already exists) gains a "assign to customer" dropdown per row + bulk-assign-and-classify action.

4. **Telemetry:** record per-session what inference path won (skill vs cwd vs transcript vs unassigned) so we can see which heuristic actually pulls weight.

**Tests**

- `eval/carver/customer-inference.test.ts` covering all four paths.
- End-to-end: end a session that included `/mitable carver` → confirm the queue row gets `customer_id_hint='carver'` → confirm scheduler auto-drains → confirm extractions land on Carver's profile.

**Honest unknowns**

- Whether `/mitable` invocations can be reliably captured. Skills don't have a "log this happened" primitive — we'd record it from the skill's own MCP-tool calls (e.g. `brief` was called with `customer='carver'` → write `recent-customer.json`).
- LLM-based transcript scan adds another `claude -p` call. Acceptable if it only runs when the cheaper heuristics fail.
- False positives. If the heuristic puts a session on the wrong customer, the profile gets polluted. Build the manual override path before lifting the noise filter.

**Out of scope**

- Multi-customer attribution within one session (a session can write to >1 customer). Defer.
- Inferring customer from MCP tool calls other than Mitable's own.

---

## Webviewer config tasks (Playbook + profile + Slack channels)

**Goal:** Make the command center a place an FDE can actually configure Mitable without dropping to the CLI / MCP tools. Right now `/sources` does Slack channels + Granola meetings well, but Playbook entries and customer profiles are read-only.

This workstream is several smallish UI tasks that share a server. Listed individually so they can be picked off.

### W1. Playbook authoring in the UI

- New page `/playbook` listing categories from `docs/05-playbook.md` weights table.
- Per category: list existing markdown files, edit-in-place textarea, save → writes to `~/.mitable/playbook/<category>/<filename>.md`.
- "New entry" button creates a file named from a slug input.
- Delete with confirmation. (Files only — never delete categories.)
- No version history in v1; user is expected to use git on `~/.mitable/playbook` if they want history.

**Backing:** new MCP tools `list_playbook`, `read_playbook_entry`, `write_playbook_entry`, `delete_playbook_entry`. Web server calls these on form submit.

### W2. Customer profile editing (FDE notes only)

The event log is append-only — this is not changing. What we **can** edit: future writes. The `/customers/<id>` page already shows the materialized profile; add a "note" composer at the bottom:

- Select profile field (dropdown of the 11)
- Textarea for content
- Submit → calls `add_note` MCP tool (already exists, v0.1.1)

Per docs/06: this is the FDE-manual write path. No editing of past entries — corrections are new events with `operation='supersede'`, which is a separate "supersede this entry" affordance worth adding later but not now.

**Backing:** existing `add_note` MCP tool. New web handler `POST /customers/:id/notes` → calls the tool internally.

### W3. Better Slack channels page

Existing `/sources` page works but:
- Doesn't show last sweep time per channel (the data is in watermarks.json — surface it)
- Doesn't show extraction counts per channel (queryable: `SELECT COUNT(*) FROM events WHERE source_ref LIKE 'C0123:%'`)
- No "test channel" button — would call `sweep_now({channel_id})` for a single channel and report what it found

**Backing:** new MCP tool `sweep_channel({channel_id, dry_run})`. New helper in `src/store/event-log.ts` for per-source counts. Templates extended to surface these columns.

### W4. Customer creation flow

`/sources` lets you map a channel to a customer, but there's no way to **create** a customer from the UI — you have to call `add_customer` manually first. Fix: 
- New page `/customers/new` with form: id, display name, one-liner, optional cwd patterns.
- Submit → `add_customer` MCP tool.
- After create, redirect to that customer's profile page with a hint to either add a channel or paste a note.

**Backing:** `add_customer` MCP tool (already exists). New web handler `GET/POST /customers/new`.

### W5. Queue assignment

Already mentioned in workstream D, repeated here for completeness:
- `/queue` page gains "assign to customer" dropdown per row
- Bulk action: "assign these N rows to <customer> and classify"

**Backing:** new MCP tool `assign_pending({session_id, customer_id})`. Existing `classify_one_session` runs after assignment.

---

## Recommended order

1. **D first.** Smallest scope, biggest immediate value, no external dependencies. Once D ships, every Claude Code session contributes to profile maintenance — that alone justifies the v0.2 bump.
2. **A second.** Real Slack adapter. Closes the largest claim-vs-reality gap. Pattern set here applies to B.
3. **W1 + W3 + W4 third.** Webviewer config tasks make A's outputs (Slack-sourced extractions) observable and editable. Pair tightly with A.
4. **B fourth.** Granola adapter, mirroring A. Smaller but valuable.
5. **W2 + W5 fifth.** Note composer + queue assignment round out the UI. Smaller items, fit between bigger workstreams.

Total surface: roughly 4 sessions of work if focused. Each numbered item above is a clean cut point — easy to stop and ship between any two.

---

## What this list deliberately leaves out

- **Stage 2 dedup (semantic).** Premature without A and B producing volume. Stage 1 is enough for the v0.1 scale.
- **Multi-FDE sync.** Per `docs/10-non-goals.md`. v2+ concern.
- **Hosted command center.** Local-only is correct per `docs/10-non-goals.md` "sensitive data on localhost".
- **Auto-built Product Manual.** Per `docs/04` the Product Manual is canonical and manually authored. The stub builder skill stays a stub.

These remain explicit non-goals until evidence pushes back.
