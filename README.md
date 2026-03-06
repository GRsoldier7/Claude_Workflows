# agentic-workflows

Production-grade n8n workflow builder using Claude Code + MCP.

**Architecture**: Brief → Spec → Node Research → Draft → Validate → Audit → Deploy

---

## Quick Start
1. `cp .env.example .env` and fill in `N8N_MCP_TOKEN`
2. `cd agentic-workflows && claude`
3. Use **Prompt 0** from `docs/prompt-library.md` to start a session
4. Write a brief in `briefs/` using `briefs/_TEMPLATE.md`
5. Use **Prompt 1** to kick off a workflow build

## Key Files
| File | Purpose |
|---|---|
| `CLAUDE.md` | Session rules — loaded every time Claude Code starts |
| `.mcp.json` | MCP server config (n8n-docs + n8n-runtime) |
| `.claude/skills/` | Reusable build skills (brief-to-workflow, validate, audit, etc.) |
| `briefs/` | One `.md` file per workflow request |
| `workflows/drafts/` | Work-in-progress workflow JSON |
| `workflows/exports/` | Versioned backups of deployed workflows |
| `docs/prompt-library.md` | Copy-paste prompts for every build phase |
| `docs/setup-guide.md` | Full setup instructions |
| `docs/deployment-checklist.md` | Deployment log + registry |

## n8n Instance
`http://100.80.13.56:5678` (Tailscale — must be on local network)
