# Setup Guide — agentic-workflows

Complete setup to go from zero to building production n8n workflows with Claude Code.

---

## Prerequisites
- Claude Code installed (`curl -fsSL https://claude.ai/install.sh | bash`)
- Node.js 18+ installed (`node --version`)
- Access to your n8n instance at `http://100.80.13.56:5678`
- n8n API key (already in `.env.example`)

---

## Step 1 — Clone / Initialize Repo

```bash
# If starting fresh:
cd ~/
git init agentic-workflows
cd agentic-workflows

# If this repo already exists, just cd in:
cd ~/agentic-workflows
```

---

## Step 2 — Set Up Environment Variables

```bash
cp .env.example .env
# .env already has your N8N_API_URL and N8N_API_KEY filled in
# You only need to add N8N_MCP_TOKEN (see Step 4)
```

---

## Step 3 — Install n8n-mcp Community Server

```bash
# Test that npx can run it (it auto-installs on first use)
npx n8n-mcp --help
```

If that fails: `npm install -g n8n-mcp`

---

## Step 4 — Enable n8n's Built-in MCP Server

1. Open `http://100.80.13.56:5678/settings/api` in your browser
2. Look for the **Instance-level MCP** section (may be under Settings > AI or Settings > MCP)
3. Enable MCP access
4. Copy the **MCP server URL** and your **personal MCP access token**
5. Add the token to `.env`:
   ```
   N8N_MCP_TOKEN=<paste token here>
   ```

> **Note**: If you don't see "Instance-level MCP" in settings, your n8n version may not support it yet.
> The community `n8n-docs` MCP (in `.mcp.json`) will still work for build-time intelligence.

---

## Step 5 — Start Claude Code

```bash
cd ~/agentic-workflows
claude
```

Inside Claude Code:
```
/login          ← if not authenticated
/mcp            ← verify MCP servers loaded (should show n8n-docs + n8n-runtime)
```

---

## Step 6 — First Session Startup

Paste this prompt at the start of your first session:

```
Read CLAUDE.md and confirm:
1. Which MCP servers are active
2. The n8n instance URL
3. Skills available in .claude/skills/
4. State of workflows/drafts/
5. Any open briefs in briefs/
```

---

## Step 7 — Build Your First Workflow

1. Copy `briefs/_TEMPLATE.md` to `briefs/<your-workflow-name>.md`
2. Fill in the brief
3. Use **Prompt 1** from `docs/prompt-library.md` in Claude Code

---

## Verify n8n API Access

```bash
curl -s \
  -H "X-N8N-API-KEY: $(grep N8N_API_KEY .env | cut -d= -f2)" \
  "http://100.80.13.56:5678/api/v1/workflows?limit=5" | jq '.data[].name'
```

If this returns workflow names, your API connection is working.

---

## Verify n8n-mcp Is Working

Inside Claude Code:
```
Use search_nodes to find the HubSpot node.
```

If it returns node information, the community MCP is connected and working.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `n8n-mcp` not found | Run `npm install -g n8n-mcp` |
| API returns 403 | Check `N8N_API_KEY` in `.env` — make sure it's the correct key from Settings > API |
| API returns "Host not allowed" | You're running from outside Tailscale — run Claude Code from your Ubuntu machine on the same network |
| MCP servers not loading | Check `.mcp.json` syntax with `cat .mcp.json | jq .` |
| n8n-runtime MCP not connecting | Verify `N8N_MCP_TOKEN` in `.env` and that Instance-level MCP is enabled in n8n settings |
