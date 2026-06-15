# Installing the Nahook MCP server

This guide tells an AI coding assistant (Cline, Claude Desktop, Cursor, Zed, …) how to
install and configure the **Nahook MCP server**. It ships as the `mcp serve` subcommand of
the `nahook` CLI — installing the CLI and authenticating once is all that's required.

## Prerequisites
- A Nahook account — sign up free at https://nahook.com (Hobby tier, no credit card).
- macOS, Linux, or Windows.

## Step 1 — Install the `nahook` CLI

**macOS / Linux (Homebrew):**
```sh
brew install getnahook/tap/nahook
```

**macOS / Linux (install script):**
```sh
curl -fsSL https://cli.nahook.com/install.sh | sh
```

Verify it's on PATH:
```sh
nahook --version
```

## Step 2 — Authenticate (once)
```sh
nahook login
```
This opens a browser for device-grant authentication and writes credentials to
`~/.nahook/config.toml`. The MCP server reads that file, so **no token needs to go into the
client config**.

## Step 3 — Register the MCP server with your client

The server runs over **stdio** via `nahook mcp serve`. Add this to your MCP client config:

```json
{
  "mcpServers": {
    "nahook": {
      "command": "nahook",
      "args": ["mcp", "serve"]
    }
  }
}
```

- **Cline:** Cline → **MCP Servers** → add the `nahook` entry above.
- **Claude Desktop / Claude Code:** `claude mcp add nahook -- nahook mcp serve`
- **Cursor:** add the block to `~/.cursor/mcp.json`.

### Optional — enable the send/trigger tools
Read tools and endpoint management work with just `nahook login`. The two tools that emit
webhooks — `trigger_event` and `send_to_endpoint` — additionally need a Nahook **ingestion
key** (`nhk_…`, the same key your SDKs use; copy it from the dashboard). Provide it via env:

```json
{
  "mcpServers": {
    "nahook": {
      "command": "nahook",
      "args": ["mcp", "serve"],
      "env": { "NAHOOK_INGESTION_KEY": "nhk_..." }
    }
  }
}
```

## Step 4 — Verify
Ask the assistant to run the **`whoami`** tool. It should return your workspace, region,
token id, and expiry. If it does, the server is connected and working.

## Available tools
- **Read:** `whoami`, `list_endpoints`, `get_endpoint`, `list_environments`,
  `list_deliveries`, `get_delivery`, `list_attempts`
- **Write (approval-gated):** `create_endpoint`, `update_endpoint`, `retry_delivery`,
  `trigger_event`, `send_to_endpoint`

Write tools carry MCP `destructiveHint`/`readOnlyHint` annotations, so your client prompts
for human approval before any state change. (Endpoints can be paused/updated but never
deleted, by design.)

## Troubleshooting
- `nahook: command not found` → ensure the CLI install dir is on your PATH; restart the
  terminal / client after installing.
- Auth or 401 errors → re-run `nahook login`.
- `trigger_event` / `send_to_endpoint` failing → set `NAHOOK_INGESTION_KEY` (see Step 3).

More: https://docs.nahook.com/mcp
