# Deploy and Host NanoClaw on Railway

NanoClaw is a personal AI assistant powered by Claude that connects to messaging channels (Slack, Telegram, Discord, WhatsApp, Gmail). It runs Claude Agent SDK in isolated processes, giving each group its own memory, skills, and tools - including web browsing, file management, and scheduled tasks.

## About Hosting NanoClaw

Deploying NanoClaw on Railway involves running a single Node.js service that spawns Claude Agent SDK processes for each incoming message. A persistent volume stores authentication state, SQLite databases, group memory, and conversation history. The service uses a multi-stage Docker build that bundles Chromium (for web browsing), the Claude Code CLI, and the agent-runner into one image.

All five channels (Slack, Telegram, Discord, WhatsApp, Gmail) are pre-installed. Set the env vars for the channels you want - channels without tokens are silently skipped at startup. No post-deploy setup commands needed.

## Common Use Cases

- Personal AI assistant accessible from your phone or desktop: ask questions, search the web, browse pages, and manage tasks
- Group-aware assistant that maintains separate memory and context per chat group, with customizable triggers and behavior
- Scheduled task automation with recurring prompts (daily summaries, reminders, monitoring) running on cron schedules

## Dependencies

- An Anthropic API key (`ANTHROPIC_API_KEY`)
- At least one channel's credentials (see [Available Channels](#available-channels) below)

### Deployment Dependencies

- [NanoClaw GitHub Repository](https://github.com/qwibitai/nanoclaw)
- [Anthropic API](https://console.anthropic.com/) for Claude access

### Core Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes | API key for Claude model access |
| `ASSISTANT_NAME` | No | Bot display name (default: `Andy`). Also used to auto-register the main group: on first startup, the bot looks for a chat matching this name and registers it as the main group. |
| `TZ` | No | Timezone for message timestamps and scheduled tasks (default: system timezone or `UTC`) |
| `GITHUB_TOKEN` | No | GitHub personal access token for installing skills from private repos |

A Railway volume must be mounted at `/data` for persistent storage (auth state, SQLite, group files).

### Auto-registration of the main group

On first startup, if no groups are registered yet, the bot syncs its chat list and looks for a group whose name matches `ASSISTANT_NAME` (case-insensitive). If found, it auto-registers that group as the main group (no trigger required, elevated privileges).

To use this:

1. Set `ASSISTANT_NAME` to your bot's name (e.g. `Andy`)
2. Create a group in your messaging app with that exact name (e.g. a WhatsApp group called "Andy")
3. Send at least one message in the group so the bot knows about it
4. Deploy (or restart) - the bot finds the group and registers it automatically

Once registered, the group persists in SQLite across restarts. This only runs when there are zero registered groups.

## Available Channels

All channels are pre-installed. Set the relevant env vars in Railway and restart - the bot auto-connects. Channels without credentials are silently skipped.

### Slack

| Variable | Description |
|----------|-------------|
| `SLACK_BOT_TOKEN` | Bot User OAuth Token (starts with `xoxb-`) |
| `SLACK_APP_TOKEN` | App-Level Token for Socket Mode (starts with `xapp-`) |
| `SLACK_MAIN_CHANNEL_ID` | (Optional) Channel ID to auto-register as the main group |

**How to get the tokens:**

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create a new app (from scratch)
2. **Socket Mode** - enable it, generate an App-Level Token (`xapp-...`) with scope `connections:write`
3. **Event Subscriptions** - enable and subscribe to bot events: `message.channels`, `message.groups`, `message.im`
4. **OAuth & Permissions** - add scopes: `chat:write`, `channels:history`, `groups:history`, `im:history`, `channels:read`, `groups:read`, `users:read`
5. **Install to Workspace** - copy the Bot User OAuth Token (`xoxb-...`)

**Setting the main channel:**

Set `SLACK_MAIN_CHANNEL_ID` to the Slack channel ID where you want to manage your bot (your private admin channel). To find the channel ID, right-click the channel in Slack > "View channel details" > the ID is at the bottom of the modal (starts with `C`). When set, this channel is auto-registered as the main group on first startup — no need to match `ASSISTANT_NAME`.

### Telegram

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Bot token from BotFather |

**How to get the token:**

1. Open Telegram and message [@BotFather](https://t.me/BotFather)
2. Send `/newbot`, follow the prompts to name your bot
3. Copy the token BotFather gives you

### Discord

| Variable | Description |
|----------|-------------|
| `DISCORD_BOT_TOKEN` | Bot token from Discord Developer Portal |

**How to get the token:**

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications) and create a new application
2. Go to **Bot** - click "Reset Token" and copy it
3. Enable **Message Content Intent** under Privileged Gateway Intents
4. Go to **OAuth2 > URL Generator** - select scopes `bot`, permissions `Send Messages` + `Read Message History` - open the generated URL to invite the bot to your server

### WhatsApp

| Variable | Description |
|----------|-------------|
| `WHATSAPP_PHONE` | Bot's phone number with country code (e.g. `+33612345678`) |

**Setup:**

The bot needs its own WhatsApp account - use a **secondary phone number**, not the one you want to message the bot from. A cheap prepaid SIM or a virtual number works fine.

1. Install **WhatsApp Business** on your phone and create an account with the secondary number
2. Set `WHATSAPP_PHONE` in Railway to that number (with country code, e.g. `+33612345678`)
3. Deploy on Railway - WhatsApp Business should send a notification to link a new device. Tap it to confirm
4. If you don't get the notification, check `railway logs` and search for "code" - you'll find an 8-digit pairing code. Enter it manually in WhatsApp Business > Settings > Linked Devices > Link a Device > Link with phone number

Auth state persists on the Railway volume. After initial pairing, the bot stays connected. `WHATSAPP_PHONE` is only needed again if the session expires.

### Gmail

| Variable | Description |
|----------|-------------|
| `GMAIL_CLIENT_ID` | OAuth 2.0 Client ID from Google Cloud Console |
| `GMAIL_CLIENT_SECRET` | OAuth 2.0 Client Secret |
| `GMAIL_REFRESH_TOKEN` | OAuth refresh token |

**How to get the credentials:**

1. Open [Google Cloud Console](https://console.cloud.google.com) - create a new project or select an existing one
2. Go to **APIs & Services > Library**, search "Gmail API", click **Enable**
3. Go to **APIs & Services > Credentials**, click **+ CREATE CREDENTIALS > OAuth client ID**
   - If prompted for consent screen: choose "External", fill in app name and email, add your email as a test user, save
   - Application type: **Desktop app**
4. Copy the **Client ID** and **Client Secret**
5. Go to [Google OAuth Playground](https://developers.google.com/oauthplayground)
   - Click the gear icon (Settings) > check **Use your own OAuth credentials** > enter your Client ID and Client Secret
   - In Step 1: find **Gmail API v1** > select `https://mail.google.com/` > click **Authorize APIs**
   - Sign in and grant access (if you see "app isn't verified", click Advanced > Go to app)
   - In Step 2: click **Exchange authorization code for tokens**
   - Copy the `refresh_token` from the response

> **Note:** If your GCP app is in "Testing" mode, refresh tokens expire after 7 days. To avoid this, go to **OAuth consent screen** and click **Publish App** (no Google review needed for personal use - only test users you added can authorize).

## Skills

Skills are reusable capabilities that extend what the agent can do — outbound email, signal detection, web scraping, etc. They're defined in GitHub repos as `SKILL.md` files and installed via [skills.sh](https://skills.sh).

### Adding skills

Tell your bot in chat:

> add skills from arnaudjnn/gtm-skills

The agent calls the `install_skills` MCP tool, which runs `npx skills add` behind the scenes, copies the skill files into the container, and reports back with any required credentials. It accepts `owner/repo` shorthand or a full GitHub URL.

### Required credentials

Some skills declare required environment variables (API keys, endpoints) in their `SKILL.md` frontmatter under `inputs`. After installing, the agent will list any missing credentials and ask you to add them in the **Railway service dashboard** (Environment Variables section). Once added, redeploy the service for them to take effect. All env vars set in Railway service config are automatically forwarded to agents — no `.env` file needed.

### Auto-update on deploy

Every time the Railway service restarts (deploy, crash recovery, manual restart), NanoClaw reads `data/skills-lock.json` from the persistent volume and re-installs all registered skill repos via `npx skills add`. This means:

- **Skills always pull the latest version** from their source repo on every deploy
- **No manual update step needed** — push changes to the skills repo, redeploy NanoClaw, done
- **Skills survive deploys** — the lock file persists on the volume even though the app directory is ephemeral

### Removing skills

To remove a skill repo, delete its entry from `data/skills-lock.json` on the persistent volume and redeploy. The skill files won't be re-installed on next startup.

### How it works under the hood

1. `install_skills` IPC task runs `npx skills add {repo} --all --copy --agent claude-code` in a temp directory
2. Installed skills are copied from the temp `.claude/skills/` into `container/skills/`
3. The repo is recorded in `data/skills-lock.json` (persistent volume)
4. At startup, `syncSkillsOnStartup()` loops through the lock file and re-runs `npx skills add` for each repo
5. `container/skills/` is synced into each group's `.claude/skills/` before every agent invocation
6. Skill `inputs` (env vars) are parsed from SKILL.md frontmatter and forwarded to agents as secrets

## Agent Capabilities

Each agent invocation has access to:

- **Tools:** Bash, Read, Write, Edit, Glob, Grep, WebSearch, WebFetch, Task management, MCP tools, Skills
- **Web browsing:** Chromium is bundled in the Docker image for screenshots and page rendering
- **Scheduled tasks:** Cron-based recurring prompts with optional pre-scripts that decide whether to wake the agent
- **Reply context:** When replying to a message, the agent sees the quoted message content and sender
- **Markdown formatting:** Outbound messages render with bold, code blocks, and links on Telegram/Slack
- **Auto-compact:** Agent sessions auto-compact at 165k tokens to prevent context overflow
- **Session cleanup:** Stale session artifacts are pruned automatically on startup and daily
- **Per-group memory:** Each group has isolated CLAUDE.md, conversation history, and session state

## Differences from the Local (Docker) Setup

The upstream [NanoClaw repository](https://github.com/qwibitai/nanoclaw) runs each agent invocation inside a Docker container, providing OS-level isolation between the host and the agent's filesystem. On Railway, Docker-in-Docker is not available, so agents run as child Node.js processes instead. Here's what changes:

| Feature | Local (Docker) | Railway |
|---------|---------------|---------|
| Agent isolation | Each agent runs in its own container with separate filesystem | Agents run as child processes sharing the host filesystem |
| Credential injection | OneCLI gateway intercepts HTTPS traffic | Local credential proxy on `127.0.0.1:3001` — agent gets a proxy URL, never the real API key |
| Filesystem sandboxing | Container mounts restrict what the agent can read/write | Directory-based separation (no OS-level enforcement) |
| Resource limits | Docker CPU/memory limits per container | Railway service-level resource limits |
| Availability | Depends on your machine being on and connected | Always on - Railway keeps the service running 24/7 |
| Network | Requires stable home internet and open ports | Railway handles networking, SSL, and uptime |

### Why This Is Fine for Personal Use

NanoClaw is designed as a **personal assistant** - you control who has access and what groups are registered. The agent already runs with `--dangerously-skip-permissions` (bypassing Claude Code's permission prompts), so container isolation is a defense-in-depth layer, not the primary security boundary. For a single-user deployment:

- **You are the only one sending prompts** - there's no untrusted input that could exploit the lack of sandboxing
- **The agent only writes to its group folder** - the Claude SDK's working directory is scoped to `/data/groups/{group}/`
- **Secrets are protected** - Anthropic API keys are handled by a local credential proxy (the agent never sees the real key). Non-Anthropic secrets (MCP vars, channel tokens) are passed via stdin with a minimal env allowlist. All secrets are stripped from Bash subprocesses by a PreToolUse hook
- **The real advantage is uptime** - Railway keeps your assistant available 24/7 without needing an always-on home machine

If you need multi-tenant isolation or expose the bot to untrusted users, consider the local Docker setup instead.

## Why Deploy NanoClaw on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying NanoClaw on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
