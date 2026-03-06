# agentic-workflows — Claude Operating Instructions

## Mission
Build production-grade n8n workflows from structured briefs.
Every workflow must be the minimum viable architecture, clearly named,
fully annotated, error-handled, validated, and safe to deploy.

---

## n8n Instance
- **URL**: `http://100.80.13.56:5678`
- **API Key**: stored in `.env` — never hardcode into workflow JSON or prompts
- **API Base**: `http://100.80.13.56:5678/api/v1`

---

## Mandatory Build Pipeline
Every single workflow request MUST follow this pipeline in order.
Do not skip steps. Do not reorder steps. Do not combine steps without confirmation.

```
1.  Read the brief          → briefs/<name>.md
2.  Search templates        → n8n-mcp: search_templates
3.  Identify nodes needed   → n8n-mcp: search_nodes
4.  Read node docs          → n8n-mcp: get_node (for each node)
5.  Produce workflow spec   → structured output (see Output Contract)
6.  Await spec approval     → confirm with user before building
7.  Build draft workflow    → n8n-mcp: n8n_create_workflow OR export JSON
8.  Validate every node     → n8n-mcp: validate_node
9.  Validate full workflow  → n8n-mcp: validate_workflow
10. Auto-fix if needed      → n8n-mcp: n8n_autofix_workflow
11. Generate test payloads  → payloads/happy-path.json + edge-case.json
12. Run workflow audit      → .claude/skills/workflow-audit/SKILL.md
13. Export JSON backup      → workflows/exports/<name>-v<n>.json
14. Return deployment steps → docs/deployment-checklist.md format
```

---

## Output Contract
For EVERY workflow request, return ALL of these before building:

| Section | Content |
|---|---|
| **Goal** | One-sentence objective |
| **Trigger** | Exact trigger type and source |
| **Required Credentials** | Name, type, scope needed |
| **Node Plan** | Ordered node list with purpose |
| **Branching Logic** | All conditional paths |
| **Error Handling** | Retry strategy + fallback for each external call |
| **Edge Cases** | List of known failure modes |
| **Open Assumptions** | Everything unconfirmed — must be flagged |
| **Test Payloads** | Happy path + at least one edge case |
| **Deployment Steps** | Manual steps user must complete in n8n |

---

## n8n Coding Standards

### Naming
- Node names MUST be descriptive: `"Fetch Lead from HubSpot"` not `"HubSpot1"`
- Workflow names: `[DOMAIN] Action - Target` e.g. `[CRM] Sync Lead - HubSpot to Airtable`
- Workflow descriptions: always fill in (purpose + trigger + required inputs + outputs)

### Node Selection
- Always prefer native n8n nodes over `HTTP Request` when a stable native node exists
- Check `search_nodes` before assuming a node name or type
- Always check node version — use latest stable unless there is a regression reason not to

### Error Handling (required on ALL external API calls)
```
External API node
  ├─ On Success → continue pipeline
  └─ On Error   → Error Handler node
                    ├─ Retry logic (exponential backoff)
                    ├─ Notification (Slack/email)
                    └─ Dead-letter logging (Airtable/Postgres)
```

### Annotations / Notes
- Every workflow MUST have a sticky note at the top explaining: purpose, trigger, inputs, outputs
- Each major section (trigger, transform, external call, output) gets its own sticky note
- Complex expressions get inline comments

### Data Handling
- Never pass raw payload forward without explicit field mapping
- Always extract only the fields you need at the earliest possible node
- Use `Set` nodes to normalize data shapes before branching

### Security
- Credentials: reference by name only — never hardcode values
- Sensitive fields: never log or pass to Slack
- Webhook paths: use non-guessable paths (UUID-style preferred)

---

## Safety Rules (non-negotiable)
1. **Never modify production directly.** Always dev first.
2. **Export backup** before any update to an existing workflow.
3. **Never delete or overwrite** a workflow unless user has explicitly confirmed.
4. **Never assume** credential names, field names, IDs, or webhook paths — always ask.
5. **Never deploy** a workflow with open assumptions unresolved.
6. **Validate before every import** — no exceptions.

---

## Skills Available
These skills live in `.claude/skills/` and auto-load when relevant.
Invoke them by name when appropriate:

| Skill | When to Use |
|---|---|
| `brief-to-workflow` | Converting a plain-English brief into a workflow spec |
| `validate-before-deploy` | Pre-deployment validation checklist |
| `workflow-audit` | Auditing an existing workflow for quality/reliability |
| `n8n-node-research` | Deep-dive research on a specific node type |
| `error-handling-patterns` | Applying standard error handling to a workflow |
| `deployment-checklist` | Step-by-step deployment guide generation |

---

## Repo Layout
```
agentic-workflows/
├─ CLAUDE.md                          ← you are here (session rules)
├─ .env                               ← N8N_API_URL + N8N_API_KEY (gitignored)
├─ .mcp.json                          ← MCP server config
├─ .claude/
│  ├─ settings.json                   ← permissions allowlist
│  └─ skills/
│     ├─ brief-to-workflow/SKILL.md
│     ├─ validate-before-deploy/SKILL.md
│     ├─ workflow-audit/SKILL.md
│     ├─ n8n-node-research/SKILL.md
│     ├─ error-handling-patterns/SKILL.md
│     └─ deployment-checklist/SKILL.md
├─ briefs/                            ← one .md per workflow request
├─ payloads/                          ← test payloads per workflow
├─ workflows/
│  ├─ drafts/                         ← working JSON (not deployed)
│  └─ exports/                        ← versioned backups
├─ docs/
│  └─ deployment-checklist.md         ← global deployment SOP
└─ .gitignore
```

---

## Session Startup Checklist
At the start of every session, confirm:
- [ ] CLAUDE.md loaded
- [ ] MCP servers active (n8n-docs + n8n-runtime)
- [ ] .env readable
- [ ] Working in `workflows/drafts/` for new work
- [ ] No production workflows will be touched without explicit confirmation
