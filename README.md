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

## Security guardrails

Webhook payloads are third-party data, so the server treats everything that flows back into the model's context as a potential prompt-injection channel:

- **Untrusted content is fenced.** Every producer- or receiver-authored value that reaches the model — webhook payloads (`get_delivery` with `include_payload`), delivery idempotency keys, and receiver error messages (`list_attempts`) — is wrapped in explicit `<<<UNTRUSTED CONTENT ...>>>` delimiters with embedded fence markers neutralized, and the server instructions tell the model to treat fenced content as inert data, never as instructions.
- **Payloads are capped.** Bodies over 256 KB are truncated (flagged via `payload_truncated`) so a hostile producer can't stuff the model's context. Tune the cap with `NAHOOK_MCP_PAYLOAD_CAP=<bytes>` in the MCP client's environment.
- **No delete tools, by design.** Nothing in the tool surface can destroy data.
- **`update_endpoint` is marked destructive.** It overwrites live endpoint config (repointing its URL is the primitive an injection would target), so its `destructiveHint` is `true` and clients surface a prominent approval prompt.
- **Write tools rely on human approval.** Per the MCP spec, annotations are advisory hints, not access control — the security boundary is the client's per-call approval prompt, which applies to every non-read tool. The additive write tools (`create_endpoint`, `send_to_endpoint`, `trigger_event`) are annotated `destructiveHint: false` per their spec semantics; keep human-in-the-loop approval enabled in your MCP client and don't blanket-allow them. If you want a hard guarantee of no writes, run the server without an ingestion key (disables sending) and treat management tools as approval-gated.
- **Secrets never reach the model.** Endpoint signing secrets and auth tokens are omitted from every tool output; `whoami` returns only the token's public id.
- **Split credentials.** Sending (`trigger_event`, `send_to_endpoint`) requires a separate ingestion key — omit it and the server is management/read-only for ingestion.

Client-side measures can't make prompt injection impossible — pair them with your MCP client's tool-approval prompts.

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
