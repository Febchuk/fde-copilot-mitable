# Mitable

A Claude Code plugin that gives AI agents the same customer, product, and operating context as a staff Forward Deployed Engineer.

See [docs/](docs/) for the full product spec. Start with [docs/01-overview.md](docs/01-overview.md).

## Status

v0.1.0 — through milestone 4. The MCP server is running with `ping`, `brief`, `seed_fixture`, and `list_customers` tools. The `/mitable <customer>` slash command is wired to the `mitable-load-context` skill. Remaining milestones: session-end hook, Slack ingest, Granola ingest, command center web UI, Playbook/Product loaders, eval harness.

## Install (local development)

```sh
# clone the repo, then:
cd fde-copilot-mitable
npm install
npm run typecheck
```

The plugin is registered via `.mcp.json` at the repo root. To load it into Claude Code, point Claude Code at this directory as a local plugin source (instructions depend on Claude Code's plugin-management UI / CLI).

On first invocation `start.sh` will:

1. `npm install` if `node_modules/` is missing
2. Run `bin/init.ts` to create `~/.mitable/` (or `$MITABLE_HOME` if set)
3. `exec` the MCP server (`src/mcp/server.ts`)

## Verify the skeleton loaded

From a Claude Code session, call the Mitable MCP `ping` tool. Expected response: `pong — mitable 0.1.0`.

## Try the Carver fixture (milestones 3 + 4)

1. Make sure `refs/carver-customer-profile/` is present (it ships in this repo, gitignored to keep design refs separate).
2. From a Claude Code session, call the `seed_fixture` MCP tool with `{ "path": "refs/carver-customer-profile" }`. Expect `{ "customer_id": "carver", "written": 11, "skipped": [...] }`.
3. Run `/mitable carver` (or `/mitable carver --mode implement`). The skill calls `brief` under the hood and tells you `Loaded carver · mode: investigate.`
4. Ask Claude anything Carver-specific. The brief is now session context — answers should reference Carver's actual workarounds, risks, and commitments.

## Layout

```
.claude-plugin/plugin.json    plugin manifest
.mcp.json                     MCP server registration
start.sh                      bootstrap + exec
bin/init.ts                   one-shot setup of ~/.mitable/
commands/mitable.md           /mitable slash command (routes to skills)
skills/                       skills the agent invokes
src/mcp/server.ts             the long-running MCP server
src/store/                    SQLite event log + fixture loader
src/assembly/                 work-mode weights + brief renderer
docs/                         canonical product spec
refs/                         original design references (gitignored)
```

## License

Apache-2.0
