---
name: mitable
description: Load customer context into the session, or (with no argument) open the Mitable command center.
---

# /mitable

The user has invoked the Mitable command. Decide which skill to delegate to based on what they passed.

## Routing

- **`/mitable <customer>` (any positional argument present)** → invoke the **`mitable-load-context`** skill. Pass the full input through; that skill knows how to parse the customer ID and optional `--mode` flag.

- **`/mitable` (no argument)** → the command center web UI. That skill ships in milestone 8 and is not yet available. Tell the user:

  > The command center UI lands in milestone 8. For now, try `/mitable <customer>` to load a customer's profile.

  Do not improvise an alternative — there's no other surface yet.

- **`/mitable --help` or anything that looks like a help request** → briefly explain:

  > `/mitable <customer>` — load that customer's profile into this session
  > `/mitable <customer> --mode investigate|implement` — pick the work mode (default: investigate)
  > `/mitable` (no arg) — open the command center (coming in milestone 8)

That's it. Don't add other behaviors here; everything else belongs in a dedicated skill.
