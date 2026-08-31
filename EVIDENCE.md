# MCP Connection + Three Tasks — Evidence

**Client:** Claude Code v2.1.251 (an MCP client)
**Server:** `@modelcontextprotocol/server-filesystem` (official reference server, stdio transport)
**Date:** 2026-08-31

---

## 1. The connection

Registered at project scope, so the config is checked into the repo and reproducible:

```jsonc
// .mcp.json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "/Users/user/.nvm/versions/node/v22.6.0/bin/npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem",
               "/Users/user/Documents/GitHub/new/ai/flyrank-mcp"],
      "env": { "PATH": "/Users/user/.nvm/versions/node/v22.6.0/bin:/usr/bin:/bin:/usr/sbin:/sbin" }
    }
  }
}
```

Health check:

```
$ claude mcp list
Checking MCP server health…
filesystem: ... @modelcontextprotocol/server-filesystem ... - ✔ Connected
```

Raw protocol handshake (proves it speaks MCP, independent of any client):

```
$ printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize",...}' ... | npx -y @modelcontextprotocol/server-filesystem <dir>
Secure MCP Filesystem Server running on stdio
{"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{"listChanged":true}},
 "serverInfo":{"name":"secure-filesystem-server","version":"0.2.0"}},"jsonrpc":"2.0","id":1}
```

Server exposes 14 tools: `read_text_file`, `read_multiple_files`, `write_file`, `edit_file`,
`list_directory`, `directory_tree`, `get_file_info`, `search_files`, `move_file`, and others.

### Method note
Every task below ran through `claude -p` with **built-in tools explicitly disabled**
(`--disallowedTools "Bash,Read,Glob,Grep,Edit,Write,WebFetch"`). This matters: it means the
results could only have come through the MCP server, not Claude Code's native file access.

---

## 2. Task 1 — read directory structure and file metadata

**Prompt:** list the `demo/` directory, then report exact byte size and mtime of `demo/inventory.csv`.

**MCP tools used:** `list_directory`, `get_file_info`

**Result:**
```
demo/ contents: config.json, inventory.csv, notes.txt
demo/inventory.csv — Size: 154 bytes — Modified: Mon Aug 31 2026, 18:45:59 GMT+0100
```

Why chat alone can't: a byte count and mtime of a file created minutes ago are not in any model's
weights and cannot be guessed.

---

## 3. Task 2 — cross-file reasoning over live local data

**Prompt:** read all three demo files, apply the threshold from `config.json` to `inventory.csv`,
cross-reference `notes.txt`, total the inventory value.

**MCP tools used:** `read_multiple_files`

**Result:**

| SKU | Item | Qty | Note |
|---|---|---|---|
| FR-2298 | rotor bearing | 4 | "stock critically low" |
| FR-4501 | brake actuator | 9 | supplier lead time extended to 6 weeks |

Total inventory value: **USD 14,467.75** (threshold 10, warehouse MSP-3, both from `config.json`).

Why chat alone can't: the arithmetic depends on three files it has never seen.

---

## 4. Task 3 — write to disk, then verify by reading back

**Prompt:** derive the below-threshold SKUs from the data, write `demo/reorder_report.md`
grounded in the real quantities and notes, then read the file back.

**Captured tool-call trace (from `--output-format stream-json`):**
```
-> TOOL CALL: mcp__filesystem__read_multiple_files  {"paths":[".../inventory.csv",".../notes.txt",".../config.json"]}
<- RESULT: {"content":"...sku,item,qty,unit_price FR-1042,hydraulic seal kit,17,84.50 ..."}
-> TOOL CALL: mcp__filesystem__write_file  {"path":".../demo/reorder_report.md","content":"# Reorder Report\n\nWarehouse: MSP-3\n..."}
<- RESULT: {"content":"Successfully wrote to .../demo/reorder_report.md"}
-> TOOL CALL: mcp__filesystem__read_text_file  {"path":".../demo/reorder_report.md"}
<- RESULT: {"content":"# Reorder Report ... Reorder threshold: 10 units (config.json) ..."}
```

`demo/reorder_report.md` now exists on disk (946 bytes) and correctly identifies FR-2298 (qty 4,
"deepest shortfall") and FR-4501 (qty 9, 6-week lead time).

Why chat alone can't: this is a **side effect outside the conversation**. The filesystem changed.

---

## 5. Negative control

The same question, with the MCP server disabled via `--strict-mcp-config` and all file/exec
tools blocked:

> "I can't read those files. The only tool available to me in this session is `ReportFindings`
> ... I won't guess at the quantity on hand for FR-2298 or the reorder threshold, since inventing
> numbers for inventory data would be worse than no answer."

Identical prompt, MCP connected → exact values. MCP disconnected → cannot answer.
That contrast is the proof the capability came from the server.

---

## Reproduce

```bash
claude mcp add -s project filesystem -- npx -y @modelcontextprotocol/server-filesystem "$PWD"
claude mcp list           # expect: ✔ Connected
claude                    # then /mcp to see the tool list
```
