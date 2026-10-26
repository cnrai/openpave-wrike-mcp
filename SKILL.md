# Wrike Skill Doctrines (SKILL.md)

Doctrines for working with Wrike from PAVE — covering the **Wrike MCP server**
(`https://mcp.wrike.com/v2`, bridged as `mcp_wrike_*` tools) and the **wrike
CLI skill** (REST v4, this package).

Every claim carries a tag. Trust them differently:

| Tag | Meaning |
|---|---|
| `[verified]` | Tested by PAVE against the live account. Unmarked = 2026-10-24; claims re-tested since carry an explicit date. |
| `[Wrike docs]` | From Wrike's own docs / tool descriptions. Not independently reproduced. |
| `[untested]` | Carried over from an earlier draft. No evidence either way. Do not rely on it. |
| `[suspect]` | Contradicts something observed. Re-test before acting on it. |

If you change something in this file, replace the blanket date with a per-claim
tag. A single "empirically verified" stamp over unverified material is worse
than no stamp, because it launders guesswork into fact.

---

## 0. Read this first — what actually breaks agents

**A. The bridge discards Wrike's error text.** `[verified 2026-10-26]`
Any Wrike `isError` surfaces through the bridge as a bare
`MCP tool reported isError` with no reason attached. Proof: `get_item_details`
with a nonexistent id returns that same bare error via the bridge, but
`Item w:itm:999999999999 not found` when the identical payload is POSTed direct
to `/v2`. Consequence: **a bare bridge error tells you nothing about why.** It
could be a bad ID, a schema-validation failure, or a dropped argument, and the
three are indistinguishable from the client side. Never infer a cause from it.
To get the real message, use the direct probe in §5.

**B. Every bridged result is returned twice.** `[verified 2026-10-26]`
Each `mcp_wrike_*` call emits the same JSON payload two times back to back,
writes included. Harmless if you read one result; wrong if you count results,
sum them, or parse the concatenation as a single document.

**C. If array arguments seem to be dropped, refresh the snapshot — do not route
around the bridge.** `[verified 2026-10-26]`
Earlier revisions of this document stated the bridge permanently drops
array-typed arguments (`itemIds`, `items`) and told agents to route every such
call through helper scripts. **That is obsolete, and it was never a permanent
defect.** It is a symptom of a stale tool snapshot: `capabilities_status`
reports `needs-review: bridged snapshot predates stored input schemas (#3244):
re-approve to restore parameter schemas`, with `bridged tools: none`. Without
the input schema the bridge does not know the parameter is an array, so it
drops it. `capabilities_refresh wrike` restores the snapshot and arrays pass.
Re-verified working **through the bridge** on 2026-10-26: `get_items_children`
(1 and 2 ids), `get_item_details`, `create_task_item` (`items[]`),
`update_items` (`itemIds[]`).
→ If arrays fail: `capabilities_refresh wrike`, confirm `bridged tools` is
populated and `needs-review` is gone, then retry. §5 is a diagnostic fallback
for reading real error text, not the default call path.

**D. Never read-modify-write a description without `fullDescriptions: true`.**
`[verified; re-confirmed 2026-10-26]` `get_item_details` truncates
`description` at **500 characters** by default (`descriptionTruncated: true`),
and `update_items.description` **replaces the entire field**. A read → tweak →
write round trip therefore silently destroys everything past 500 chars, with no
error and no warning. Measured on a real task: default read returned 500 chars,
`fullDescriptions: true` returned 1721. Note `descriptionTruncated: false` only
means the stored value fit the limit — it is not evidence the write was
complete.

---

## 1. Installing the Wrike MCP in PAVE

Wrike MCP v2 is a remote Streamable-HTTP server. `[verified]`

```
capabilities_add https://mcp.wrike.com/v2
```

First connect fails `Unauthorized` (status `error`) — expected. Wrike requires
auth and publishes **no OAuth discovery metadata**. All three probes confirm
the dead end: `[verified]`

- unauthenticated `POST /v2` → `401`, and the response carries **no**
  `WWW-Authenticate` header
- `/.well-known/oauth-protected-resource` → `404`
- `/.well-known/oauth-authorization-server` → `404`

**Why OAuth 2.0 cannot complete.** Wrike's authorization server requires a
**confidential client** (client secret). PAVE's MCP OAuth is a **public PKCE
client with no secret by design** (ADR-031; `mcp-oauth-flow.js` refuses
confidential ASs before any state is minted). Wrike's docs direct non-DCR
clients to a Permanent Access Token. `[Wrike docs]` for the AS requirement;
the discovery probes above are `[verified]`.

### PAT installation steps (the supported path)

1. Generate a token: Wrike → profile icon → **Apps & Integrations** → **API**
   tab → open/create an app (e.g. "Wrike MCP Integration") → **Permanent
   access token** → *Create token*. The token inherits the creating user's
   permissions (use Raymond's account).
2. Store it as `MCP__wrike__ACCESS: <token>` in `~/.pave/tokens.yaml` (0600).
   Sanctioned alternative: `PUT /capabilities/mcp/wrike/tokens` with body
   `{"access": "<token>"}` on the PAVE server (owner-gated) — it also pins
   `audienceOrigin` for you. `[untested — endpoint shape never exercised]`
3. Ensure the registry entry (`~/.pave/mcp/servers.json`) carries
   `auth.type: "bearer"` (not `none`) and `audienceOrigin: https://mcp.wrike.com`.
   `[verified]` `capabilities_add` writes `auth.type: "none"`, so **this edit is
   mandatory** — the token is never attached without it.
4. Sync the CLI skill: set `WRIKE_ACCESS_TOKEN` to the same value (one token
   serves both MCP and REST).
5. Reconnect: `capabilities_refresh wrike` → status `installed`, 26 tools.
   `[verified]`
6. **Approve first use.** `[verified]` The first `mcp_wrike_*` call fails with
   *"requires approval on first use (R10 default: ask)"*. That is not a broken
   install. Resolve it with `permission(action="grant", type="mcp",
   target="mcp:wrike")` — the user sees a confirmation card — then retry.
7. Verify: MCP initialize probe with the Bearer token returns HTTP 200
   `[verified]`; `GET /api/v4/account` with the same token returns 200
   `[untested this session]`.

> **Never re-run `capabilities_add` to repair a broken connection.** `[verified]`
> It rewrites the registry entry with `auth.type: "none"`, destroying the bearer
> config and the `audienceOrigin` pin. The repair verb is `capabilities_refresh`.

Tokens live ONLY in `tokens.yaml` — never in `servers.json`, process env, or
bash (R6). Never echo token values into output. Helper scripts must read the
token from the file, never accept it on the command line.

---

## 2. Calling the tools: parameter names

`[verified; re-confirmed 2026-10-26]` against live `tools/list`. Dump real
schemas with
`NODE_USE_ENV_PROXY=1 node ~/.pave/cache/wrike-probe.mjs schema <tool>`.
Whether you can inspect them offline depends on snapshot freshness: a snapshot
that predates stored input schemas shows only an `inputSchemaHash` per tool and
leaves `bridged tools: none` (see §0C). All parameter names below were
re-checked against `tools/list` on 2026-10-26 and are correct — array failures
are never a naming problem.

| Tool | Parameters |
|---|---|
| `get_item_details` | `itemIds` (array, 1–5), `fullDescriptions` (bool) |
| `get_items_children` | `itemIds` (array, 1–25), `limitPerParent` (int, 1–100, default 25) |
| `update_items` | `itemIds` (array, 1–20) + fields: `title`, `description`, `status`, `customStatus`, `importance`, `dates{}`, … |
| `create_task_item` | `items` (array of `{parentId*, title*, description, assigneeId, customItemTypeId, startDate, dueDate, durationMinutes, status}`) |
| `search_items` | `query` — **not** `keyword` |
| `get_users` | exactly one of `ids`, `emails`, `me` |

`*` required. `status` enum: `ACTIVE` / `DEFERRED` / `COMPLETED` / `CANCELLED`.
`importance` enum is **uppercase**: `HIGH` / `NORMAL` / `LOW` — lowercase
`normal` is rejected by server-side schema validation. `[verified 2026-10-26]`
`durationMinutes`: 1–1,799,999 working minutes; 480 = one Wrike working day.

All four array tools above work through the bridge. `[verified 2026-10-26]` Use
§5 only to read a real error message when a call fails and the bridge has
swallowed the reason (§0A), or to inspect schemas — not as the default path.

Unverified: `importance` sent on `create_task_item` did not appear in the create
response echo or in any subsequent read, so it is unknown whether it applied.
The create response echoes only `parentId` and `title` regardless of what is
sent. `[untested]`

---

## 3. Description formatting doctrine

> MCP `create_task_item` schema, verbatim:
> `description: "Optional HTML description. Convert requested plain text to
> HTML without adding content; omit if unspecified."`

Rules are identical for MCP and REST v4 — same underlying field. `[verified]`

**What survives the sanitizer**

- **Kept:** `<b> <i> <u>` — including nesting (`<b><i>`, `<b><u>`, `<i><u>` all
  round-trip intact) · `<br>` · `<ul> <ol> <li>` · `<a href>` ·
  **`<table> <tr> <td>`**
- **Stripped:** `<th>` — the tags are removed and the header text collapses into
  one run-on cell (`<tr>TenderClosesStatus</tr>`). **All attributes** are
  dropped, including `border="1"`.
- **Headings** (`<h1>`–`<h6>`): not in the kept set. `[untested]`

> **Tables work.** An earlier revision of this document said "no tables in the
> basic set." That was wrong. Use `<td><b>Header</b></td>` for header rows —
> never `<th>`.

**Markdown is never parsed, and newlines vanish without a trace.** `[verified]`
Sent `LINE_A_ONE\nLINE_A_TWO\n**bold markdown** and # heading and - bullet`,
stored as:

```
LINE_A_ONELINE_A_TWO**bold markdown** and # heading and - bullet- second bullet
```

Note the failure mode is worse than "dropped": `\n` disappears with **no
separator**, so adjacent lines fuse into one word. Every line break must be an
explicit `<br/>`.

**Server-side escaping** `[verified]`

- `+` → `&#43;` · backtick → `` &#96; `` · `&` → `&amp;`
- `=` **inside an `href`** → `&#61;` — harmless; browsers decode entities in
  attribute values, so the link still resolves.
- Named entities are decoded to literals on save (`&middot;` → `·`).
- `<br/>` is normalised to `<br />`, so a read-back diff against your input will
  always show cosmetic churn. Don't treat that as corruption.

**Agent pipeline:** build descriptions as HTML directly — bold section labels,
`<br/>` separators, `<ul>/<ol>` for lists, `<a>` for links, `<table>` with
`<td><b>…</b></td>` headers. Do not ship markdown into Wrike.

---

## 4. ID doctrine

- MCP IDs: `w:spc:N` (space), `w:itm:N` (folder/project/task), `w:usr:N`,
  `w:ct:N`, `w:rf:N`, `w:acj:S`. `[verified]`
- **`open.htm?id=N` ≡ `w:itm:N`.** `[verified]` This is string concatenation,
  not a lookup — the `get_item_details` and `get_items_children` schemas both
  say *"Build it, don't search."* A full permalink is also accepted and resolved
  automatically.
- REST v4 accepts **canonical IDs only** (e.g. `MAAAAAERp_30`, `MQAAAAEOOTDU`);
  numeric permalink IDs are rejected (`"Invalid Task ID"`). `[untested this
  session — no REST calls made]`
- Numeric → canonical resolution (REST only): list the parent's children via
  REST and match by title. Canonical objects carry a `permalink` with the
  numeric id, so the reverse direction is free. `[untested]`
- ~~Tasks cannot be created in a space root~~ **`[suspect]`** — the
  `create_task_item` schema explicitly accepts `parentId` as `w:spc:N` for a
  space. Never verified; may be a stale note from an older API behaviour.
  Re-test before refusing a space-root create.
- Folder creation via REST requires a canonical parent path; `POST /folders`
  with a space parameter is refused. `[untested]`

---

## 5. Diagnostic fallback: drive the server directly

Use this to read a real error message the bridge has swallowed (§0A), or to dump
schemas. It is **not** the default call path — array arguments work through the
bridge (§0C). Same token, Streamable HTTP:

1. `POST https://mcp.wrike.com/v2` — JSON-RPC `initialize`, then
   `notifications/initialized`, then `tools/list` / `tools/call`, with
   `Authorization: Bearer <PAT>` and
   `Accept: application/json, text/event-stream`.
2. **Framing:** observed `content-type: application/json` on both `tools/list`
   and `tools/call`. `[verified]` An earlier revision said responses "may be
   SSE-framed" — keep a defensive parser that takes the last `data:` line if one
   is present, but do not expect SSE or block waiting for a session id.
   `tools/list` uses **chunked transfer-encoding**; frame it by bytes, not
   string indices, or multi-byte UTF-8 in tool descriptions desynchronises the
   chunk offsets. `[verified 2026-10-26]`
3. No `mcp-session-id` is returned; the server tolerates its absence. `[verified]`
4. **`NODE_USE_ENV_PROXY=1` is required** in this container. Node 22's `fetch`
   ignores `HTTPS_PROXY` without it, and egress to `mcp.wrike.com` only works
   through the proxy. `[verified 2026-10-26]`

Helpers — token-safe, reading the PAT from `tokens.yaml`, never argv:

```bash
NODE_USE_ENV_PROXY=1 node ~/.pave/cache/wrike-probe.mjs tools/list
NODE_USE_ENV_PROXY=1 node ~/.pave/cache/wrike-probe.mjs schema get_item_details
NODE_USE_ENV_PROXY=1 node ~/.pave/cache/wrike-call.mjs call get_items_children '{"itemIds":["w:itm:123"]}'
NODE_USE_ENV_PROXY=1 node ~/.pave/cache/wrike-call.mjs call create_task_item @args.json   # @file for big HTML
```

**These files are not in this repository and `~/.pave/cache/` is not
version-controlled — they get wiped.** `[verified 2026-10-26]` They were found
missing twice, which silently made the workaround documented in earlier
revisions of this file unrunnable. Check they exist before relying on them;
`wrike-call.mjs` is a symlink to `wrike-probe.mjs`.

Reads and writes also work via REST v4 (`https://www.wrike.com/api/v4`,
canonical IDs, same Bearer token). `[untested this session]`

---

## 6. MCP toolset facts

- **26 tools, fixed set.** `[verified]` **No delete operations** — destructive
  actions are not exposed through MCP at all (aligns with house rules: deletion
  needs Raymond's explicit approval anyway). Consequence: **anything you create
  as a test must be deleted by hand in the Wrike UI.** Say so before creating
  test artifacts in shared folders.
- `create_task_item` takes an `items[]` array (bulk). Response per item:
  `itemId` + `permalink`. `[verified]`
- Retry discipline `[Wrike docs]` (from the tool description): after a mixed
  response retry only failed items; if the outcome is unknown, search the parent
  for an exact-title match and ask before retrying. Never invent IDs.
- Clients cache the tool list; after Wrike ships new tools, re-run
  `capabilities_refresh` to re-snapshot. `[Wrike docs]`
- **Every result is returned twice**, writes included. `[verified 2026-10-26]`
  Parse one payload; do not count or concatenate.
- `capabilities_refresh` is also the repair for **dropped array arguments** — a
  stale snapshot has no input schemas, so the bridge cannot tell an array from a
  scalar. Refresh before diagnosing, and never re-run `capabilities_add`.
  `[verified 2026-10-26]`
- `create_task_item` echoes only `parentId` and `title` in its `input` field,
  regardless of which fields you sent. A successful create is therefore not
  evidence that `description`, `importance`, or `status` applied — read the item
  back. `[verified 2026-10-26]`

---

## 7. Config reference

| Key | Where | Purpose | Status |
|---|---|---|---|
| `MCP__wrike__ACCESS` | `~/.pave/tokens.yaml` | MCP bearer token (0600) | `[verified]` |
| `WRIKE_ACCESS_TOKEN` | `~/.pave/tokens.yaml` | CLI skill REST token (same PAT) | `[untested]` |
| `WRIKE_HOST` | `~/.pave/tokens.yaml` | API host override (default `www.wrike.com`) | `[untested]` |
| `auth.type: bearer` | `~/.pave/mcp/servers.json` | attach token on every MCP call | `[verified]` |
| `audienceOrigin` | `~/.pave/mcp/servers.json` | R6 pin; URL changes never replay the token | `[verified]` |
| `mcp -> mcp:wrike` | `~/.pave/permissions.yaml` | R10 first-use approval for the bridge | `[verified]` |

Token expired? Both MCP and REST return 401 — regenerate the PAT once, update
both keys (step 2 and 4 above). Tokens belong to the generating user; Wrike
permissions apply end-to-end.

---

## 8. Open questions

Carry these as known-unknowns rather than quiet assertions:

1. Space-root task creation — allowed by schema, asserted impossible here. `[suspect]`
2. REST v4 canonical-ID strictness and the `/folders` space refusal. `[untested]`
3. `PUT /capabilities/mcp/wrike/tokens` request/response shape. `[untested]`
4. Whether `<table>` renders as a visual grid in the Wrike **web UI**. All table
   evidence here is *stored HTML on read-back*; nobody has looked at the browser.
5. Heading tags (`<h1>`–`<h6>`) — assumed stripped, never probed.
6. Does `importance` on `create_task_item` actually apply? Sent `LOW`, never
   seen back on read. `[untested]`
7. Are the two bridge defects in §0A/§0B reported upstream? Swallowed error text
   and doubled results are PAVE-side bugs, not Wrike's, and the first one is
   what let a stale snapshot masquerade as a permanent "bridge drops arrays"
   defect for two days.
