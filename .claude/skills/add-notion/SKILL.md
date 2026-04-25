---
name: add-notion
description: Add Notion MCP integration to NanoClaw on Railway. Wires up the official @notionhq/notion-mcp-server via .mcp.json with NOTION_TOKEN forwarded from Railway env vars, and patches the Railway runner to forward MCP secrets into the agent-runner child process. Use when the user wants the assistant to read or edit their Notion workspace (pages, databases, tasks).
---

# Add Notion Integration

Adds Notion as an MCP server so the agent can manage tasks, pages, and databases in the user's Notion workspace. Uses Notion's official internal-integration token flow (no OAuth, headless-friendly).

This skill also fixes a gap in NanoClaw's Railway runner where MCP secret values are read from stdin but never reach the agent-runner's `process.env`. The patch makes any future MCP server work too, not just Notion.

## Phase 1: Pre-flight

### Check if already applied

```bash
test -f .mcp.json && grep -q '"notion"' .mcp.json && echo "notion-already-in-mcp-json" || echo "needs-mcp-json"
grep -q '\.\.\.filteredSecrets' src/railway-runner.ts && echo "runner-already-patched" || echo "needs-runner-patch"
```

If both report "already", skip to Phase 3 (Setup) — the user just needs to set `NOTION_TOKEN` in Railway and redeploy.

### Confirm with the user

Tell the user this skill will:
1. Add `.mcp.json` at the repo root with the Notion server entry.
2. Patch `src/railway-runner.ts` to forward MCP secrets to the agent-runner.
3. Walk them through getting a Notion integration token and setting it in Railway.

Ask if they want to proceed. If yes, continue.

## Phase 2: Apply Code Changes

### Add or merge `.mcp.json`

If `.mcp.json` does not exist, create it:

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "NOTION_TOKEN": "${NOTION_TOKEN}"
      }
    }
  }
}
```

If `.mcp.json` already exists, read it, merge in the `notion` entry under `mcpServers`, and write it back. Do not overwrite existing servers.

The `${NOTION_TOKEN}` placeholder is required — `src/container-runner.ts:collectMcpEnvVars()` scans `.mcp.json` for `${VAR}` references to know which Railway env vars to forward.

### Patch `src/railway-runner.ts`

The current Railway runner spawns the agent-runner with an explicit `env` block that omits MCP secrets, and the agent-runner reads secret values from `process.env[key]` rather than from the stdin `secrets` payload. Result: `NOTION_TOKEN` (and any other MCP secret) never reaches the MCP server.

Find this block (around line 158):

```typescript
  const logsDir = path.join(groupDir, 'logs');
  fs.mkdirSync(logsDir, { recursive: true });

  return new Promise((resolve) => {
    const child = spawn('node', [agentRunnerPath], {
      stdio: ['pipe', 'pipe', 'pipe'],
      cwd: groupDir,
      env: {
        PATH: process.env.PATH || '/usr/local/bin:/usr/bin:/bin',
        // ...
        ANTHROPIC_BASE_URL: 'http://127.0.0.1:' + CREDENTIAL_PROXY_PORT,
        ANTHROPIC_API_KEY: 'proxy-injected',
      },
    });

    onProcess(child, processName);

    let stdout = '';
    let stderr = '';
    let stdoutTruncated = false;
    let stderrTruncated = false;

    // Pass non-Anthropic secrets via stdin (Anthropic auth handled by credential proxy)
    const allSecrets = readSecrets();
    const ANTHROPIC_KEYS = ['ANTHROPIC_API_KEY', 'CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_AUTH_TOKEN', 'ANTHROPIC_BASE_URL'];
    const filteredSecrets: Record<string, string> = {};
    for (const [k, v] of Object.entries(allSecrets)) {
      if (!ANTHROPIC_KEYS.includes(k)) filteredSecrets[k] = v;
    }
    input.secrets = filteredSecrets;
```

Replace with:

```typescript
  const logsDir = path.join(groupDir, 'logs');
  fs.mkdirSync(logsDir, { recursive: true });

  // Read non-Anthropic secrets up front so we can both forward them into the
  // child process env (so MCP servers can resolve ${VAR} placeholders) and
  // also pass them via stdin for backward compatibility with the legacy path.
  // Anthropic auth is handled separately via the credential proxy.
  const ANTHROPIC_KEYS = ['ANTHROPIC_API_KEY', 'CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_AUTH_TOKEN', 'ANTHROPIC_BASE_URL'];
  const allSecrets = readSecrets();
  const filteredSecrets: Record<string, string> = {};
  for (const [k, v] of Object.entries(allSecrets)) {
    if (!ANTHROPIC_KEYS.includes(k)) filteredSecrets[k] = v;
  }

  return new Promise((resolve) => {
    const child = spawn('node', [agentRunnerPath], {
      stdio: ['pipe', 'pipe', 'pipe'],
      cwd: groupDir,
      env: {
        PATH: process.env.PATH || '/usr/local/bin:/usr/bin:/bin',
        NODE_PATH: process.env.NODE_PATH || '',
        TZ: TIMEZONE,
        HOME: claudeDir.replace(/\/.claude$/, ''), // Parent of .claude dir
        NANOCLAW_WORKSPACE_GROUP: groupDir,
        NANOCLAW_WORKSPACE_GLOBAL: globalDir || '',
        NANOCLAW_WORKSPACE_EXTRA: extraDir || '',
        NANOCLAW_IPC_DIR: ipcDir,
        NANOCLAW_IPC_INPUT: path.join(ipcDir, 'input'),
        LOG_LEVEL: process.env.LOG_LEVEL || '',
        NODE_ENV: process.env.NODE_ENV || '',
        RAILWAY_ENVIRONMENT: process.env.RAILWAY_ENVIRONMENT || '',
        ANTHROPIC_BASE_URL: 'http://127.0.0.1:' + CREDENTIAL_PROXY_PORT,
        ANTHROPIC_API_KEY: 'proxy-injected',
        // Forward MCP / skill / channel secrets so the agent-runner can
        // populate sdkEnv from process.env[key] and MCP servers receive
        // resolved values for ${VAR} placeholders in .mcp.json.
        ...filteredSecrets,
      },
    });

    onProcess(child, processName);

    let stdout = '';
    let stderr = '';
    let stdoutTruncated = false;
    let stderrTruncated = false;

    // Also pass secrets via stdin (legacy path; harmless when env is set above)
    input.secrets = filteredSecrets;
```

Use the `Edit` tool (single replacement; the block is unique).

### Validate

```bash
npm install
npm run build
```

The build must succeed. If TypeScript complains, re-read the patched section and confirm the spread syntax (`...filteredSecrets`) sits inside the `env` object literal.

If unit tests exist for the runner, run:

```bash
npx vitest run src/railway-runner
```

Tests should still pass.

## Phase 3: Setup

### Notion integration token

Tell the user:

1. Open `https://www.notion.so/profile/integrations`.
2. Click **New integration**, give it a name like "NanoClaw", pick the workspace, select **Internal** integration type. Save.
3. Under **Configuration**, set **Capabilities** to whatever they want the agent to do. For full task management: read content, update content, insert content, read user info without email. (For least privilege, start read-only and expand later.)
4. Copy the **Internal Integration Secret** (starts with `ntn_`).
5. Connect the integration to the pages and databases NanoClaw should access:
   - Open the target page or database in Notion.
   - Click the three-dots menu (top right) → **Connections** → search for the integration name → **Connect**.
   - For databases: connecting the database parent page is enough; child pages inherit access.

The integration will only see what's explicitly connected. This is the security boundary.

### Set the Railway env var

Tell the user:

1. Open the Railway project → NanoClaw service → **Variables**.
2. Add `NOTION_TOKEN` with the `ntn_...` value from step 4 above.
3. Save. Railway will redeploy automatically.

### Wait for redeploy

Once the redeploy is live, the user should ask the assistant in their main chat something like:

```
list my notion tasks
```

or

```
create a notion page called "Test" under the Inbox database
```

If it works, the agent will use `mcp__notion__*` tools transparently.

## Troubleshooting

If Notion tools don't appear:

```bash
# Check the agent-runner log for the MCP load line
LOG_LEVEL=debug
# Look for: "MCP server from .mcp.json: notion (npx)"
```

Common failures:

- **Notion returns 401 / "API token is invalid"** — the integration was deleted or the token was rotated. Generate a new one in Notion and update Railway.
- **"object_not_found" when accessing a specific page** — the integration isn't connected to that page yet. Open the page in Notion, three-dots menu → Connections → connect the integration.
- **No `mcp__notion__*` tools at all** — `.mcp.json` didn't make it into the agent's `.claude` dir. Confirm `.mcp.json` is at the repo root (not inside `src/` or `container/`), and check that `prepareWorkspace` in `src/railway-runner.ts` still has the line `fs.copyFileSync(mcpJsonSrc, path.join(claudeDir, '.mcp.json'))`.
- **Tools appear but every call hangs** — `npx -y @notionhq/notion-mcp-server` is downloading the package on first run. Give it 30 seconds. Subsequent calls hit the npm cache (kept on the Railway volume because `HOME` points inside `/data`).

## Notes

- This uses Notion's stdio MCP server. The remote OAuth-based server at `mcp.notion.com` requires an interactive browser flow on first launch and persisting OAuth state in `~/.mcp-auth/`, which is fragile on a headless Railway box. The integration-token route is what Notion's own README recommends for self-hosted clients.
- The runner patch is generic. Adding more MCP servers later (Linear, Figma, Sentry, etc.) is now just: append to `.mcp.json` with `${SOME_TOKEN}`, set `SOME_TOKEN` in Railway, redeploy.
- If the user already added `.mcp.json` manually before running this skill, the merge step preserves their existing entries.
