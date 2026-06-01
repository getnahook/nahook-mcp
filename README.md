# nahook-mcp

The official **Model Context Protocol** server for [Nahook](https://nahook.com) — drive webhook deliveries, endpoints, and event triggers from Claude Desktop, Cursor, Cline, and any other MCP-compatible AI assistant.

> The MCP server ships as a subcommand of the [`nahook` CLI](https://github.com/getnahook/nahook-cli). One binary, one credentials file, one install path.

## What it does

Once installed and authenticated, your AI assistant can:

- **Inspect** endpoints, deliveries, and per-attempt logs to debug failed webhooks.
- **Retry** failed or dead-lettered deliveries.
- **Trigger** events and fan them out to every subscribed endpoint.
- **Send** webhooks directly to a single endpoint.
- **Manage** endpoints (create, update, pause, resume) — never delete, by design.

Write operations are tagged with the MCP `destructiveHint`/`readOnlyHint` annotations so MCP clients surface a per-call human-approval prompt before anything mutates state.

## Install

The MCP server is a subcommand of the `nahook` CLI. Install the CLI, then add it to your AI client.

### 1. Install the CLI

**macOS / Linux (Homebrew):**

```sh
brew install getnahook/tap/nahook
```

**Linux / macOS (install script):**

```sh
curl -fsSL https://cli.nahook.com/install.sh | sh
```

See [getnahook/nahook-cli](https://github.com/getnahook/nahook-cli) for other install options.

### 2. Log in once

```sh
nahook login
```

This opens your browser, walks you through device-grant authentication, and writes credentials to `~/.nahook/config.toml`.

### 3. Wire the MCP server into your AI client

**Claude Desktop & Claude Code CLI:**

```sh
claude mcp add nahook -- nahook mcp serve
```

**Cursor** — edit `~/.cursor/mcp.json`:

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

**Cline (VS Code)** — Cline → Settings → MCP Servers → add:

```json
{
  "nahook": {
    "command": "nahook",
    "args": ["mcp", "serve"]
  }
}
```

**Zed** — add to `settings.json`:

```json
{
  "assistant": {
    "mcp_servers": {
      "nahook": {
        "command": "nahook",
        "args": ["mcp", "serve"]
      }
    }
  }
}
```

Other MCP-compatible clients (Continue, Windsurf, Goose, …) follow the same `command + args` shape.

## Tools

| Tool | What it does |
|------|--------------|
| `whoami` | Local config sanity check — workspace, region, token id, expiry. |
| `list_endpoints` | List every endpoint in the current workspace. |
| `get_endpoint` | Fetch one endpoint by `ep_xxx`. |
| `create_endpoint` | Create a new endpoint. Defaults to the workspace's default environment; also accepts slugs like `production`. |
| `update_endpoint` | Partial patch — pause/resume, change URL, update description. |
| `list_environments` | List every environment in the workspace. |
| `list_deliveries` | Page through an endpoint's deliveries, newest-first. |
| `get_delivery` | Fetch one delivery by `del_xxx`. Pass `include_payload: true` to also fetch the original webhook body — critical for debugging. |
| `list_attempts` | List every attempt against a delivery (useful for debugging failures). |
| `retry_delivery` | Re-enqueue a failed or dead-lettered delivery. |
| `trigger_event` | Fire an event by type — the backend fans it out to every subscriber. |
| `send_to_endpoint` | Send a webhook directly to one endpoint. |

## Authentication model

The MCP server uses two separate credentials:

- **CLI login token** (`nhc_…`) — written by `nahook login`. Powers read tools and management operations on endpoints / deliveries / environments.
- **Ingestion key** (`nhk_…`) — the same key your SDKs use. Powers `trigger_event` and `send_to_endpoint`. Set it via `NAHOOK_INGESTION_KEY=nhk_...` in the MCP client's environment, or add `ingestion_key = "nhk_..."` to `~/.nahook/config.toml`. The server fails loudly if a write tool is called without one configured.

Permissions are evaluated **per request** against your workspace role — the MCP token has exactly the permissions you have in the dashboard, nothing more.

## Where the code lives

The MCP server implementation is part of the [`nahook` CLI source tree](https://github.com/getnahook/nahook-cli/tree/main/internal/commands/mcp). This repo exists as a discoverability surface — README, MCP directory entries, and packaging metadata. There is no separate codebase to maintain.

If you want to file a bug, request a tool, or contribute: please open the issue at [getnahook/nahook-cli](https://github.com/getnahook/nahook-cli/issues).

## License

[MIT](./LICENSE) — same as the CLI.
