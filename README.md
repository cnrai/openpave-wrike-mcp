# wrike-mcp

Doctrine skill for working with Wrike from PAVE via the remote **Wrike MCP
server** (`https://mcp.wrike.com/v2`, bridged as `mcp_wrike_*` tools).

This is a doctrine-only skill: it ships no local CLI. Its payload is
[`SKILL.md`](SKILL.md) — verified operational knowledge for the Wrike MCP
bridge, including:

- Known bridge defects (e.g. array-typed MCP arguments failing with a bare
  `isError`) and their workarounds
- The `fullDescriptions: true` read-modify-write trap (descriptions truncate
  at 500 chars by default; `update_items.description` replaces the whole
  field)
- Installing the Wrike MCP in PAVE (`capabilities_add
  https://mcp.wrike.com/v2`)

## Install

```bash
pave install cnrai/openpave-wrike-mcp
```

Then read the doctrine from any PAVE session:

```
skill_docs(skill="wrike-mcp")
```

## Relationship to `openpave-wrike`

`cnrai/openpave-wrike` is the standalone REST-v4 CLI skill (token-based,
`wrike_*` commands). This skill is its MCP-bridged counterpart: no local
commands, no tokens — everything goes through the remote MCP server.

## License

MIT
