# Moltworker Complete Setup Guide

Deploy OpenClaw (formerly Moltbot/Clawdbot) on Cloudflare Workers using Sandbox containers.

> **Note**: Moltworker is a proof-of-concept. It is not officially supported and may break without notice.

---

## Prerequisites

- **Cloudflare Workers Paid plan** ($5/month minimum) - required for Sandbox containers
- **Anthropic API key** for Claude access, OR Cloudflare AI Gateway configuration
- **Node.js** and **npm** installed locally
- **Wrangler CLI** (installed via npm)

---

## Step 1: Clone and Install

```bash
git clone https://github.com/cloudflare/moltworker.git
cd moltworker
npm install
```

---

## Step 2: Set Required Secrets

### AI Provider (pick one)

**Option A: Direct Anthropic API**
```bash
npx wrangler secret put ANTHROPIC_API_KEY
# Paste your Anthropic API key when prompted
```

**Option B: Cloudflare AI Gateway**
```bash
npx wrangler secret put AI_GATEWAY_API_KEY
npx wrangler secret put AI_GATEWAY_BASE_URL
# Format: https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_id}/anthropic
```

AI Gateway provides caching, rate limiting, and analytics. Create a gateway in the AI Gateway section of the Cloudflare dashboard, add Anthropic as a provider.

### Gateway Token (required for Control UI)
```bash
export MOLTBOT_GATEWAY_TOKEN=$(openssl rand -hex 32)
echo "Save this token: $MOLTBOT_GATEWAY_TOKEN"
npx wrangler secret put MOLTBOT_GATEWAY_TOKEN
```

---

## Step 3: Deploy

```bash
npm run deploy
```

Access the Control UI at: `https://your-worker.workers.dev/?token=YOUR_GATEWAY_TOKEN`

---

## Step 4: Set Up Cloudflare Access (Required for Admin UI)

The admin UI at `/_admin/` requires Cloudflare Access protection.

### Via Workers Dashboard
1. Go to Workers & Pages dashboard
2. Select your Moltworker Worker → Settings
3. Under Domains & Routes, enable Cloudflare Access on `workers.dev`
4. Configure access policy (add email or identity provider)
5. Copy the Application Audience (AUD) tag

### Set Access Secrets
```bash
npx wrangler secret put CF_ACCESS_TEAM_DOMAIN
# Enter your team domain (e.g., your-team.cloudflareaccess.com)

npx wrangler secret put CF_ACCESS_AUD
# Paste the AUD tag from step above

npm run deploy
```

### Alternative: Zero Trust Dashboard
Create a Self-hosted application in Cloudflare Zero Trust Dashboard protecting:
- `/_admin/*`
- `/api/*`
- `/debug/*`

---

## Step 5: Device Pairing

1. Navigate to `/_admin/` (authenticate via Cloudflare Access)
2. Open the Control UI from another device/browser
3. New devices appear as "pending" in the admin UI
4. Approve devices you want to grant access

This is the default "pairing" policy - the most secure option requiring explicit approval.

---

## Optional: Persistent Storage (R2)

Without R2, all data is lost on container restart.

### Create R2 Bucket and API Token
1. Go to R2 Overview in Cloudflare Dashboard
2. Create a bucket named `moltbot-data`
3. Click "Manage R2 API Tokens"
4. Create token with **Object Read & Write** permissions for `moltbot-data`
5. Copy Access Key ID and Secret Access Key

### Set R2 Secrets
```bash
npx wrangler secret put R2_ACCESS_KEY_ID
npx wrangler secret put R2_SECRET_ACCESS_KEY
npx wrangler secret put CF_ACCOUNT_ID
npm run deploy
```

### How R2 Persistence Works
- **Startup**: R2 data restores to the OpenClaw config directory
- **Runtime**: Cron job syncs config to R2 every 5 minutes
- **Admin UI**: Shows last backup timestamp; has "Backup Now" button

---

## Optional: Chat Platform Integrations

### Telegram
```bash
npx wrangler secret put TELEGRAM_BOT_TOKEN
npm run deploy
```

### Discord
```bash
npx wrangler secret put DISCORD_BOT_TOKEN
npm run deploy
```

### Slack
```bash
npx wrangler secret put SLACK_BOT_TOKEN
npx wrangler secret put SLACK_APP_TOKEN
npm run deploy
```

---

## Optional: Browser Automation (CDP)

Enables headless Chrome control via Chrome DevTools Protocol.

```bash
npx wrangler secret put CDP_SECRET
# Enter a secure random string

npx wrangler secret put WORKER_URL
# Enter: https://your-worker.workers.dev

npm run deploy
```

### Built-in Browser Skills

**Screenshots:**
```bash
node /root/clawd/skills/cloudflare-browser/scripts/screenshot.js https://example.com output.png
```

**Video from multiple URLs:**
```bash
node /root/clawd/skills/cloudflare-browser/scripts/video.js "https://site1.com,https://site2.com" output.mp4 --scroll
```

---

## Optional: Container Lifecycle

Default: container stays alive indefinitely (`SANDBOX_SLEEP_AFTER=never`). This is recommended since cold starts take 1-2 minutes.

To auto-sleep after inactivity:
```bash
npx wrangler secret put SANDBOX_SLEEP_AFTER
# Enter: 10m, 30m, 1h, etc.
```

With R2 configured, data persists across sleep/wake cycles.

---

## Debug Endpoints

Available when `DEBUG_ROUTES=true` and Cloudflare Access is enabled:

| Endpoint | Description |
|----------|-------------|
| `GET /debug/processes` | List container processes |
| `GET /debug/logs?id=<pid>` | View process logs |
| `GET /debug/version` | Version information |

---

## Local Development

Create a `.dev.vars` file in the project root:
```
DEV_MODE=true
DEBUG_ROUTES=true
```

This bypasses Cloudflare Access and device pairing for local testing.

```bash
npm run dev
```

---

## Complete Secrets Reference

| Secret | Required | Purpose |
|--------|----------|---------|
| `ANTHROPIC_API_KEY` | Yes* | Claude API key |
| `ANTHROPIC_BASE_URL` | No | Custom Anthropic endpoint |
| `AI_GATEWAY_API_KEY` | Yes* | AI Gateway provider key |
| `AI_GATEWAY_BASE_URL` | Yes* | AI Gateway endpoint URL |
| `OPENAI_API_KEY` | No | Alternative AI provider |
| `MOLTBOT_GATEWAY_TOKEN` | Yes | Control UI authentication |
| `CF_ACCESS_TEAM_DOMAIN` | Yes** | Cloudflare Access team domain |
| `CF_ACCESS_AUD` | Yes** | Access app audience tag |
| `R2_ACCESS_KEY_ID` | No | R2 persistent storage |
| `R2_SECRET_ACCESS_KEY` | No | R2 persistent storage |
| `CF_ACCOUNT_ID` | No | Required if using R2 |
| `TELEGRAM_BOT_TOKEN` | No | Telegram integration |
| `DISCORD_BOT_TOKEN` | No | Discord integration |
| `SLACK_BOT_TOKEN` | No | Slack integration |
| `SLACK_APP_TOKEN` | No | Slack integration |
| `CDP_SECRET` | No | Browser automation auth |
| `WORKER_URL` | No | Worker public URL (for CDP) |
| `SANDBOX_SLEEP_AFTER` | No | Container sleep timeout |

\* At least one AI provider required (Anthropic API or AI Gateway)
\** Required for admin UI security

---

## Troubleshooting

### Container takes long to start
Cold starts take 1-2 minutes. Keep `SANDBOX_SLEEP_AFTER=never` (default) for always-on operation.

### Data lost after restart
Configure R2 storage (see above). Without R2, container storage is ephemeral.

### Can't access admin UI
Ensure Cloudflare Access is configured and `CF_ACCESS_TEAM_DOMAIN` / `CF_ACCESS_AUD` secrets are set.

### New device can't send messages
Devices require admin approval via the `/_admin/` panel before they can interact.

---

## Resources

- [Moltworker GitHub](https://github.com/cloudflare/moltworker)
- [Cloudflare Sandbox SDK Docs](https://developers.cloudflare.com/sandbox/)
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [Cloudflare Blog: Moltworker](https://blog.cloudflare.com/moltworker-self-hosted-ai-agent/)
