---
name: cloudflare-mcp
description: Guide for deploying and running AI agents (OpenClaw/Moltbot/Clawdbot) on Cloudflare using the Container Sandbox MCP server. Use when setting up Cloudflare sandbox containers, deploying Moltworker, configuring MCP-based remote code execution environments, or running always-on AI assistants on Cloudflare Workers.
license: Apache-2.0
---

# Cloudflare MCP & Moltworker Deployment Guide

Deploy AI agents on Cloudflare using the Container Sandbox MCP server and Moltworker (OpenClaw/Moltbot/Clawdbot).

---

## Two Approaches

### 1. Cloudflare Container Sandbox MCP Server
A remote MCP server for executing code in sandboxed containers. Ideal for quick prototyping, code execution, and lightweight sandbox use.

**URL**: `https://containers.mcp.cloudflare.com/mcp`

### 2. Moltworker (OpenClaw on Cloudflare)
A full deployment of OpenClaw (formerly Moltbot/Clawdbot) running inside Cloudflare Sandbox containers. Ideal for always-on personal AI assistants with chat platform integrations.

**Repo**: `https://github.com/cloudflare/moltworker`

---

## Approach 1: Cloudflare Container Sandbox MCP

### What It Does
Provides sandboxed container environments where LLMs can execute code (Python, Node.js, shell), manage files, and perform operations in isolation. Containers are ephemeral (~10-minute lifespan).

### MCP Client Configuration

For clients that support remote MCP servers natively, use:
```
https://containers.mcp.cloudflare.com/mcp
```

For clients using `mcp-remote` (e.g., Claude Desktop):
```json
{
  "mcpServers": {
    "cloudflare-sandbox": {
      "command": "npx",
      "args": ["mcp-remote", "https://containers.mcp.cloudflare.com/mcp"]
    }
  }
}
```

Authentication happens via Cloudflare OAuth in the browser on first connection.

### Available Tools

| Tool | Description |
|------|-------------|
| `container_initialize` | Start or restart a container |
| `container_ping` | Test container connectivity |
| `container_exec` | Run shell commands |
| `container_file_write` | Create or modify files |
| `container_file_read` | Read file or directory contents |
| `container_files_list` | List working directory contents |
| `container_file_delete` | Remove files or directories |

### Example Use Cases
- Run Python/Node.js scripts in isolation
- Clone and test repositories
- Generate visualizations with matplotlib
- Perform data analysis
- Test code snippets without local setup

---

## Approach 2: Moltworker (OpenClaw on Cloudflare Workers)

Moltworker deploys OpenClaw as an always-on AI assistant inside Cloudflare Sandbox containers. It provides multi-platform chat integration, persistent storage, and browser automation.

For complete setup instructions, see: [Moltworker Setup Guide](./reference/moltworker-setup.md)

### Prerequisites
- Cloudflare Workers Paid plan ($5/month minimum)
- Anthropic API key (or Cloudflare AI Gateway)
- Node.js and npm installed locally

### Quick Start

```bash
# Clone the repo
git clone https://github.com/cloudflare/moltworker.git
cd moltworker
npm install

# Set required secrets
npx wrangler secret put ANTHROPIC_API_KEY
export MOLTBOT_GATEWAY_TOKEN=$(openssl rand -hex 32)
npx wrangler secret put MOLTBOT_GATEWAY_TOKEN

# Deploy
npm run deploy
```

Access the Control UI at: `https://your-worker.workers.dev/?token=YOUR_GATEWAY_TOKEN`

### Architecture

```
                    Cloudflare Workers
┌──────────────────────────────────────────────┐
│  Gateway Worker                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Telegram  │  │ Discord  │  │   Slack   │  │
│  │   Bot     │  │   Bot    │  │    Bot    │  │
│  └─────┬─────┘  └────┬─────┘  └─────┬─────┘  │
│        └──────────────┼──────────────┘        │
│                  ┌────▼────┐                  │
│                  │ Control │                  │
│                  │   UI    │                  │
│                  └────┬────┘                  │
│              ┌────────▼────────┐              │
│              │ Sandbox Container│              │
│              │  ┌────────────┐ │              │
│              │  │  OpenClaw  │ │              │
│              │  │  (Claude)  │ │              │
│              │  └────────────┘ │              │
│              │  ┌────────────┐ │              │
│              │  │ R2 Storage │ │              │
│              │  │ (optional) │ │              │
│              │  └────────────┘ │              │
│              └─────────────────┘              │
└──────────────────────────────────────────────┘
```

### Key Features
- **Multi-platform**: Telegram, Discord, Slack, and web Control UI
- **Device pairing**: Secure authentication requiring admin approval
- **Persistent storage**: Optional R2 storage preserves data across restarts
- **Browser automation**: Chrome DevTools Protocol (CDP) support
- **Always-on**: Container stays alive indefinitely by default
- **AI Gateway**: Optional Cloudflare AI Gateway for caching, rate limiting, analytics

### Security Layers
1. **Cloudflare Access** - Protects admin routes (`/_admin/`, `/api/*`, `/debug/*`)
2. **Gateway Token** - Required for Control UI access
3. **Device Pairing** - Admin approval required for each new device

### Container Lifecycle
Default: container stays alive indefinitely (`SANDBOX_SLEEP_AFTER=never`).

To configure auto-sleep:
```bash
npx wrangler secret put SANDBOX_SLEEP_AFTER
# Enter: 10m, 30m, 1h, etc.
```

Cold starts take 1-2 minutes. R2 storage preserves data across restarts.

---

## Choosing an Approach

| Criteria | Container Sandbox MCP | Moltworker |
|----------|----------------------|------------|
| Setup complexity | Minimal (just add MCP config) | Moderate (deploy Worker) |
| Use case | Code execution, prototyping | Always-on AI assistant |
| Persistence | Ephemeral (~10 min) | Persistent with R2 |
| Chat integrations | None (MCP only) | Telegram, Discord, Slack |
| Cost | Free tier available | $5/month Workers Paid |
| Browser automation | No | Yes (CDP) |

---

## Reference Files

- [Moltworker Setup Guide](./reference/moltworker-setup.md) - Complete deployment and configuration
- [Cloudflare Sandbox SDK](https://developers.cloudflare.com/sandbox/) - Platform documentation
- [MCP Server Repo](https://github.com/cloudflare/mcp-server-cloudflare/tree/main/apps/sandbox-container) - Container sandbox MCP source
- [Moltworker Repo](https://github.com/cloudflare/moltworker) - OpenClaw on Cloudflare source
