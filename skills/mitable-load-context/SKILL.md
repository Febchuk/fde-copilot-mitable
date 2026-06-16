---
name: mitable-load-context
description: Load a customer's profile into the session as background context. Use when the user runs /mitable <customer>, asks you to "load context for X", or asks anything customer-specific where you don't yet have the customer's profile in this session.
---

# Mitable — Load Customer Context

You give the user the experience of starting every customer-facing conversation pre-briefed. The mechanism is one MCP call.

## When to invoke

- The user typed `/mitable <customer>` (or `/mitable <customer> --mode <mode>`)
- The user asked you to "load Carver" / "pull up the Acme profile" / similar
- You're about to answer a customer-specific question and don't have that customer's profile already loaded in this session

If the user typed `/mitable` with **no argument**, this is NOT the right skill — that opens the command center (different skill, milestone 8). For now, tell them: "the command center UI ships in milestone 8 — try `/mitable <customer>` to load a profile."

## What to do

1. **Parse the customer ID and (optionally) the work mode** from the user's input.
   - Customer ID is the first positional argument. Lowercase it. Example: `/mitable Carver` → `carver`.
   - Mode is `--mode investigate` or `--mode implement`, defaulting to `investigate` when omitted.

2. **Call the `brief` tool** on the Mitable MCP server:
   ```
   mcp__mitable__brief({
     customer: "<customer_id>",
     mode: "<mode>"
   })
   ```
   The exact tool name in your tool list will be whatever Claude Code has registered the Mitable MCP under — typically `mcp__mitable__brief` or similar. Look for the tool whose description says it renders the customer-context brief.

3. **If the brief comes back saying "No profile entries yet for this customer"**, tell the user that customer is not yet seeded. Suggest:
   - For the worked example (Carver): "Run `seed_fixture` against `refs/carver-customer-profile` first."
   - For real customers: "Configure their Slack channels in the command center to populate the profile via automatic ingestion."

4. **Otherwise, treat the returned markdown as authoritative context for this session.** Specifically:
   - Acknowledge the load with one terse line: `Loaded <customer> · mode: <mode>.` Do NOT echo the brief back at the user — it's already on screen.
   - From this point in the conversation, all your answers about this customer should be informed by the brief.
   - Treat entries tagged `measured` and `customer_reported` as higher trust than `inferred`.
   - When citing specifics, mention the source type and date inline ("per Slack on 2025-05-20…") so the user can audit your reasoning.

## What NOT to do

- Don't paraphrase or summarize the brief unsolicited. The user already sees it.
- Don't call `brief` again in the same session unless the user explicitly asks for a refresh or switches customer/mode. Briefs are snapshots; re-fetching mid-session is noise.
- Don't extract anything from the brief into long-term memory or files. The event log is the only source of truth.
- Don't make up customer IDs. If the user types a customer you can't find, call `list_customers` and offer the closest match — don't guess.

## Examples

**User:** `/mitable carver`
**You:** *(call `brief({customer: "carver", mode: "investigate"})`)*
**You:** `Loaded carver · mode: investigate.`

**User:** `/mitable Carver --mode implement`
**You:** *(call `brief({customer: "carver", mode: "implement"})`)*
**You:** `Loaded carver · mode: implement.`

**User:** `/mitable acme`
**You:** *(call `brief({customer: "acme"})`)* → returns "No profile entries yet"
**You:** `Acme isn't seeded yet. Configure their Slack channels via the command center (ships in milestone 8) or seed a fixture manually with the seed_fixture MCP tool.`
