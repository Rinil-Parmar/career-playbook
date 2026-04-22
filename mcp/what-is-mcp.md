# MCP — Model Context Protocol

**One line:** Standardized protocol so any LLM can talk to any external tool or data source — like USB-C but for AI agents.

---

## The Problem It Solves

Before MCP, every AI integration was custom:
- Claude → your DB: custom code
- Claude → GitHub: different custom code
- GPT → same DB: write it again from scratch

Multiply that by 100 tools × 10 LLMs = unmaintainable mess. MCP standardizes the connection layer so you write the server once and any LLM can use it.

Anthropic open-sourced MCP in late 2024. Handed control to the Linux Foundation in late 2025. Now adopted by AWS, OpenAI, Red Hat, and most major AI platforms.

---

## Architecture

```
┌─────────────────┐        MCP Protocol         ┌──────────────────────┐
│   LLM Host      │ ◄──────────────────────────► │    MCP Server        │
│ (Claude Code,   │   JSON-RPC over stdio/HTTP   │  (your tool, DB,     │
│  Cursor, GPT)   │                              │   API, file system)  │
└─────────────────┘                              └──────────────────────┘
        │
        │ manages
        ▼
┌─────────────────┐
│   MCP Client    │
│ (built into     │
│  the host app)  │
└─────────────────┘
```

**Three components:**

| Component | Role | Example |
|-----------|------|---------|
| MCP Host | The LLM app | Claude Code, Cursor, your app |
| MCP Client | Lives inside host, manages connections | Built into Claude |
| MCP Server | Exposes tools/data via MCP protocol | Your Go server |

---

## What an MCP Server Exposes

Three primitives:

```
Tools      → functions the LLM can CALL     (read_file, query_db, send_email)
Resources  → data the LLM can READ          (file contents, DB rows, API responses)
Prompts    → reusable prompt templates      (optional, for common workflows)
```

---

## Transport Layer

**stdio** — local process, most common. Claude Code uses this.
```
Host spawns server as child process → communicates over stdin/stdout
```

**HTTP + SSE** — remote server, for networked tools.
```
Host connects to HTTP endpoint → server streams responses via Server-Sent Events
```

---

## Minimal MCP Server in Go

Official Go SDK: `github.com/mark3labs/mcp-go`

```go
package main

import (
    "context"
    "os"

    "github.com/mark3labs/mcp-go/mcp"
    "github.com/mark3labs/mcp-go/server"
)

func main() {
    s := server.NewMCPServer("bank-server", "1.0.0")

    tool := mcp.NewTool("get_balance",
        mcp.WithDescription("Get account balance for a given account ID"),
        mcp.WithString("account_id", mcp.Required(), mcp.Description("The account ID")),
    )

    s.AddTool(tool, func(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
        accountID := req.Params.Arguments["account_id"].(string)
        // replace with real DB/API call
        return mcp.NewToolResultText("Balance for " + accountID + ": $1,234.56"), nil
    })

    server.NewStdioServer(s).Listen(context.Background(), os.Stdin, os.Stdout)
}
```

---

## Minimal MCP Server in Python

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types

server = Server("bank-server")

@server.list_tools()
async def list_tools():
    return [types.Tool(
        name="get_balance",
        description="Get account balance",
        inputSchema={
            "type": "object",
            "properties": {
                "account_id": {"type": "string", "description": "The account ID"}
            },
            "required": ["account_id"]
        }
    )]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_balance":
        account_id = arguments["account_id"]
        return [types.TextContent(type="text", text=f"Balance for {account_id}: $1,234.56")]

stdio_server(server)
```

Install: `pip install mcp`

---

## Connecting to Claude Code

Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "bank-server": {
      "command": "/path/to/your/bank-server",
      "args": []
    }
  }
}
```

Claude can now call your tools directly in conversation.

---

## Why MCP Matters for Fintech Co-op

Canada's Open Banking framework is rolling out in 2025-2026. Banks will expose standardized APIs for account data, transactions, and balances. An MCP server that wraps these APIs means:

- Any LLM can query a user's financial data with natural language
- Privacy-first: can run fully local with Ollama
- First-mover advantage: no open-source Canadian Open Banking MCP server exists yet

**Project idea:** Build `mcp-openbanking-ca` in Go — an MCP server for Canadian Open Banking APIs. This is a Go project (your strength), timely (regulation just launching), and visible (GitHub stars from Canadian fintech devs).

---

## Resources

- Official spec: [modelcontextprotocol.io](https://modelcontextprotocol.io)
- Go SDK: [github.com/mark3labs/mcp-go](https://github.com/mark3labs/mcp-go)
- Python SDK: `pip install mcp`
- MCP Developers Summit (2026): [x.com/mcpsummit](https://x.com/mcpsummit)

*Last updated: 2026-04-22*
