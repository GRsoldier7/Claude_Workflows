I couldn’t pull the full transcript, so I rebuilt this from the video’s public snippets plus the official Claude Code, MCP, and n8n documentation. The pattern shown is clear: stop hand-building every workflow node-by-node, use Claude Code as the build interface, give it MCP access to n8n knowledge and optionally your live n8n instance, and keep reusable rules in `CLAUDE.md` plus skills. The video’s own framing is “type the outcome, let Claude write, run, hit errors, fix them, and keep going.” ([LinkedIn][1])

Here is the best-practice setup I would use.

## 1) Install Claude Code

Claude Code supports macOS 13+, Windows 10 1809+/Server 2019+, Ubuntu 20.04+, Debian 10+, and Alpine 3.19+, and Anthropic’s recommended install method is the native installer. Claude Code starts in any project folder with `claude`. ([Claude API Docs][2])

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows PowerShell
irm https://claude.ai/install.ps1 | iex
```

After install:

```bash
mkdir agentic-workflows
cd agentic-workflows
claude
```

Use `/login` if you are not already authenticated. ([Claude API Docs][3])

## 2) Create one dedicated repo for workflow building

Use a repo that is only for workflow design, prompts, exported JSON, test payloads, and deployment notes. Claude Code loads project instructions from `CLAUDE.md` every session, can use project-scoped MCP servers from `.mcp.json`, and auto-discovers skills from `.claude/skills/.../SKILL.md`. ([Claude API Docs][4])

Use this structure:

```text
agentic-workflows/
├─ CLAUDE.md
├─ .mcp.json
├─ .claude/
│  ├─ settings.json
│  └─ skills/
│     ├─ brief-to-workflow/
│     │  └─ SKILL.md
│     ├─ validate-before-deploy/
│     │  └─ SKILL.md
│     └─ workflow-audit/
│        └─ SKILL.md
├─ briefs/
│  └─ example-brief.md
├─ payloads/
│  ├─ happy-path.json
│  └─ edge-case.json
├─ workflows/
│  ├─ drafts/
│  └─ exports/
└─ docs/
   └─ deployment-checklist.md
```

## 3) Put the permanent rules in `CLAUDE.md`

Anthropic recommends using `/init` to generate a starter `CLAUDE.md`, then refining it. Their guidance is to keep `CLAUDE.md` short, use it for rules Claude cannot infer on its own, and move occasional procedures into skills so you do not bloat every session. ([Claude API Docs][5])

Run:

```bash
claude
/init
```

Then replace the generated file with something like this:

```md
# Mission
Build n8n workflows from business briefs with the fewest possible nodes,
clear names, strong validation, and safe deployment steps.

# Required build order
1. Convert the brief into a workflow spec.
2. Search existing n8n templates before designing from scratch.
3. Search nodes and read node docs before choosing parameters.
4. Produce a draft workflow design.
5. Validate nodes and the full workflow before exporting or deploying.
6. Never assume credential names or IDs.
7. Never modify production first.

# n8n standards
- Prefer native nodes over HTTP Request when the native node exists and is stable.
- Use explicit node names like "Fetch Leads from HubSpot" not generic names like "HubSpot1".
- Add error-handling branches for every external API call.
- Add comments / notes describing purpose, inputs, and outputs.
- Keep triggers simple and downstream logic modular.

# Output contract
For every workflow request, return:
1. workflow goal
2. trigger
3. required credentials
4. node-by-node plan
5. edge cases
6. test payloads needed
7. deployment steps

# Safety
- Work in dev first.
- Export a backup before any destructive update.
- Never delete or overwrite a workflow unless explicitly instructed.
```

That file becomes the operating system for how Claude behaves in this repo. ([Claude API Docs][4])

## 4) Enable n8n’s built-in MCP server on your instance

n8n has an official built-in MCP server. It lets MCP clients search eligible workflows, retrieve workflow metadata, and trigger exposed workflows. To enable it, go to `Settings > Instance-level MCP`, turn on MCP access, then copy the server URL and your personal MCP access token from the Connection details panel. ([n8n Docs][6])

Important constraints from n8n:

* Only **published** workflows are eligible.
* Eligible workflows must contain a **Webhook, Schedule, Chat, or Form** trigger.
* No workflows are exposed by default; you must explicitly enable MCP access per workflow.
* n8n recommends adding a workflow description so MCP clients can understand what inputs the workflow expects.
* MCP-triggered executions have a **5-minute timeout**.
* Binary input is not supported; inputs are text-based. ([n8n Docs][6])

## 5) Connect Claude Code directly to n8n’s official MCP server

n8n’s docs now include a Claude Code connection example. The official command is: ([n8n Docs][6])

```bash
claude mcp add --transport http n8n-mcp https://<your-n8n-domain>/mcp-server/http \
  --header "Authorization: Bearer <YOUR_N8N_MCP_TOKEN>"
```

This gives Claude Code runtime access to exposed workflows in your n8n instance. Use it for discovery, metadata lookup, and execution of already-published workflows. ([n8n Docs][6])

## 6) Add the community `n8n-mcp` server for build-time intelligence

This is the missing piece that makes the video’s workflow really powerful. The community `czlonkowski/n8n-mcp` server is not n8n’s built-in MCP runtime; it is a separate MCP server that gives Claude deep knowledge of n8n nodes, node properties, operations, templates, examples, and validation tools. With API configuration, it can also create, update, validate, list, and auto-fix workflows on your instance. ([GitHub][7])

Its high-value tools include `search_nodes`, `get_node`, `validate_node`, `validate_workflow`, `search_templates`, and `get_template`. With `N8N_API_URL` and `N8N_API_KEY`, it also exposes management tools like `n8n_create_workflow`, `n8n_update_full_workflow`, `n8n_update_partial_workflow`, `n8n_list_workflows`, and `n8n_autofix_workflow`. ([GitHub][7])

Use this project-scoped `.mcp.json`:

```json
{
  "mcpServers": {
    "n8n-runtime": {
      "type": "http",
      "url": "https://<your-n8n-domain>/mcp-server/http",
      "headers": {
        "Authorization": "Bearer <YOUR_N8N_MCP_TOKEN>"
      }
    },
    "n8n-docs": {
      "command": "npx",
      "args": ["n8n-mcp"],
      "env": {
        "MCP_MODE": "stdio",
        "LOG_LEVEL": "error",
        "DISABLE_CONSOLE_OUTPUT": "true",
        "N8N_API_URL": "https://<your-n8n-domain>",
        "N8N_API_KEY": "<YOUR_N8N_PUBLIC_API_KEY>"
      }
    }
  }
}
```

Claude Code documents `.mcp.json` as the standard format for project-scoped MCP servers, and project-scoped servers require approval before use. The `n8n-mcp` project documents the `npx n8n-mcp` configuration and the required env vars for management tools. ([Claude API Docs][8])

One constraint: n8n’s public REST API is not available during the free trial, so if you do not have API access yet, leave out `N8N_API_KEY` and use `n8n-docs` only for research/validation, then import JSON manually into n8n. ([n8n Docs][9])

## 7) Add Claude Code permissions so you are not approving every tiny action

Claude Code requests permission by default for edits, bash commands, and MCP tools. Anthropic recommends using `/permissions` allowlists or sandboxing to reduce approval fatigue while keeping control. ([Claude API Docs][5])

Create `.claude/settings.json`:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff)",
      "Bash(ls *)",
      "Bash(cat *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  }
}
```

Keep this conservative. Do not give broad shell permissions until the repo is stable. Anthropic explicitly warns that skipping permissions entirely should only be done in a sandboxed environment. ([Claude API Docs][5])

## 8) Create three skills, not one giant prompt

Anthropic’s skills system works by placing a `SKILL.md` file in `.claude/skills/<skill-name>/`. Claude auto-discovers these skills and can invoke them when relevant, which is much cleaner than keeping one giant reusable prompt in chat history. ([Claude API Docs][10])

Create these skills.

### A. `brief-to-workflow/SKILL.md`

```md
# brief-to-workflow

Turn a plain-English automation brief into an n8n workflow spec.

## Process
1. Read the brief.
2. Identify trigger, inputs, outputs, external systems, credentials, and failure cases.
3. Search existing n8n templates first.
4. Search nodes needed for each step.
5. Produce:
   - workflow objective
   - trigger
   - required credentials
   - node-by-node design
   - branching logic
   - retries / error handling
   - test payloads
   - open assumptions

## Rules
- Prefer the smallest viable workflow.
- Prefer native n8n nodes over generic HTTP calls.
- Never invent credentials, IDs, or field names.
- Flag all assumptions explicitly.
```

### B. `validate-before-deploy/SKILL.md`

```md
# validate-before-deploy

Validate an n8n workflow before deployment.

## Process
1. Validate each chosen node.
2. Validate the full workflow.
3. Check trigger fit, credential requirements, field mappings, pagination, rate limits, and error paths.
4. Output:
   - blocking issues
   - non-blocking improvements
   - missing credentials
   - missing test payloads
   - go / no-go recommendation
```

### C. `workflow-audit/SKILL.md`

```md
# workflow-audit

Audit an existing workflow for maintainability and production readiness.

## Check for
- unclear node names
- duplicated logic
- brittle mappings
- missing retries
- missing fallback branches
- hidden assumptions
- poor notes / documentation
- places where a template or native node would be better

## Output
Return:
1. top risks
2. exact fixes
3. simplified redesign if needed
```

## 9) Use this exact operating loop every time you build a workflow

This is the closest thing to the repeatable framework shown by the video.

### Step 9.1: Write the brief first

Put every request in `briefs/<name>.md` with this format:

```md
# Goal
When a new Typeform response arrives, create or update a HubSpot contact,
classify lead quality, notify Slack if score > 80, and add the record to Airtable.

# Trigger
Typeform submission

# Inputs
- name
- email
- company
- role
- budget
- timeline
- freeform notes

# Outputs
- HubSpot contact updated
- lead score assigned
- Slack alert for high intent
- Airtable row created

# Constraints
- avoid duplicates
- retry on HubSpot rate limits
- no Slack alert if email missing
- store raw payload
```

### Step 9.2: Start Claude Code in the repo

```bash
cd agentic-workflows
claude
```

First prompt:

```text
Read CLAUDE.md and confirm which MCP servers are available.
Then read briefs/typeform-hubspot-airtable.md.
Use the brief-to-workflow skill.
Search existing templates first.
Then search the exact nodes required.
Do not build yet; return the workflow spec, required credentials,
unknown assumptions, and the smallest recommended architecture.
```

That prompt forces Claude into the right order: spec first, then template search, then node search, then architecture. This is where most people save the most time. ([Claude API Docs][5])

### Step 9.3: Make Claude turn the spec into a build plan

Second prompt:

```text
Now convert the approved spec into an n8n implementation plan.
For each node, give:
- node name
- node type
- purpose
- key parameters
- expected input data
- expected output data
- error handling path
Then identify which parts must be validated with validate_node
and which assumptions still need confirmation.
```

At this stage, you want design clarity, not JSON yet.

### Step 9.4: Build in one of two ways

Use **Path A** if your `n8n-docs` MCP has API access. Use **Path B** if it does not.

**Path A — direct creation/update through MCP:** ask Claude to create the draft workflow in your dev n8n instance using the management tools, then retrieve the created workflow and validate it. The community MCP exposes creation, update, list, validation, and auto-fix tools specifically for this. ([GitHub][7])

**Path B — JSON export/import:** ask Claude to generate workflow JSON into `workflows/drafts/<name>.json`, then import it into n8n either from the Editor UI (`Import from File` or `Import from URL`) or via the CLI command `n8n import:workflow --input=file.json`. n8n officially documents both the UI import path and the CLI import command. ([n8n Docs][11])

Use this prompt for Path B:

```text
Generate the workflow JSON and save it as
workflows/drafts/typeform-hubspot-airtable.json.

Before finalizing:
1. validate all nodes
2. validate the workflow
3. list any fields that still require manual credential binding in n8n
4. create two test payload files:
   - payloads/happy-path.json
   - payloads/edge-case.json
```

## 10) Do not let Claude guess n8n node details

This is the main reason the community `n8n-mcp` server matters. Its stated value is deep node documentation, properties, operations, examples, template search, and validation, precisely to avoid the common “guessing game” around n8n conventions. ([GitHub][7])

Whenever Claude seems uncertain, force this sequence:

```text
Search the relevant nodes.
Read the node docs.
Compare versions if needed.
Validate the selected node parameters.
Only then update the workflow.
```

That single habit will prevent most broken workflows.

## 11) Add workflow descriptions inside n8n for future Claude runs

n8n’s MCP docs specifically say workflow descriptions help MCP clients identify workflows and understand expected inputs, especially for webhook-style workflows. After each workflow is working, write a description in n8n that includes purpose, trigger, required input fields, and output behavior. ([n8n Docs][6])

Use this template in the n8n workflow description:

```text
Purpose: Ingest Typeform submissions and create/update HubSpot contacts,
score lead quality, notify Slack for high-intent leads, and log to Airtable.

Trigger: Typeform webhook
Required inputs: name, email, company, role, budget, timeline, notes
Returns: contact status, lead score, Airtable row info
Constraints: no Slack message if email missing; retry HubSpot rate limits
```

This makes future agent sessions much better because Claude can discover and use workflows more reliably. ([n8n Docs][6])

## 12) Keep dev and prod separate

The community `n8n-mcp` project explicitly warns not to edit production workflows directly with AI, and recommends copying workflows, testing in development first, exporting backups, and validating changes before deployment. ([GitHub][7])

Use this deployment rule:

1. Build in dev.
2. Validate in dev.
3. Export JSON backup.
4. Promote to prod.
5. Publish.
6. Re-enable MCP exposure on the published version if needed, because MCP eligibility in n8n is based on the published workflow and access must be explicitly enabled. ([n8n Docs][6])

## 13) The exact prompt I would use for almost every new workflow

```text
Read CLAUDE.md.
Use the brief-to-workflow skill on briefs/<name>.md.

Process in this order:
1. search templates
2. search nodes
3. read docs for selected nodes
4. propose smallest viable workflow architecture
5. list required credentials
6. flag all assumptions
7. build draft workflow
8. validate nodes
9. validate full workflow
10. save JSON to workflows/drafts/<name>.json
11. generate happy-path and edge-case payloads
12. run workflow-audit
13. return:
   - summary
   - node-by-node design
   - validation findings
   - manual steps I must complete in n8n
```

That prompt is strong because it forces Claude to behave like a workflow engineer, not a free-form chatbot.

## 14) The best “leverage” pattern from this framework

Use this division of labor:

* `CLAUDE.md` = permanent repo rules.
* Skills = repeatable procedures.
* Official n8n MCP = runtime discovery and execution of exposed workflows.
* Community `n8n-mcp` = build-time research, node docs, template search, validation, and optional workflow management.
* n8n itself = stable production runtime. ([Claude API Docs][4])

That combination is the most complete version of the workflow shown by the video, because it gives Claude both **knowledge of how to build n8n workflows** and **access to the actual n8n environment**.

## 15) Do this first, in order

1. Install Claude Code. ([Claude API Docs][3])
2. Create the repo structure above. ([Claude API Docs][4])
3. Run `/init`, then replace `CLAUDE.md` with the version above. ([Claude API Docs][5])
4. Enable `Settings > Instance-level MCP` in n8n and copy your MCP URL/token. ([n8n Docs][6])
5. Add the official n8n MCP server to Claude Code. ([n8n Docs][6])
6. Add the community `n8n-mcp` server to `.mcp.json`. ([Claude API Docs][8])
7. Add the three skills. ([Claude API Docs][10])
8. Build the first workflow from a brief, not from ad hoc chat.
9. Validate before every import or update. ([GitHub][7])
10. Promote only after dev testing and backup export. ([GitHub][7])

The most important practical change is this: stop asking Claude to “make a workflow,” and instead make it follow a fixed pipeline of **brief → spec → template search → node research → draft → validation → deploy**. That is where the speed and consistency actually come from.

[1]: https://www.linkedin.com/posts/jonocatliff_%F0%9D%90%88-%F0%9D%90%AB%F0%9D%90%9E%F0%9D%90%A9%F0%9D%90%A5%F0%9D%90%9A%F0%9D%90%9C%F0%9D%90%9E%F0%9D%90%9D-%F0%9D%90%A6%F0%9D%90%B2-%F0%9D%90%A78%F0%9D%90%A7-%F0%9D%90%B0%F0%9D%90%A8%F0%9D%90%AB%F0%9D%90%A4%F0%9D%90%9F-activity-7434313812687663104-7RGW?utm_source=chatgpt.com "Replacing n8n with Claude for automation"
[2]: https://docs.anthropic.com/en/docs/claude-code/setup?utm_source=chatgpt.com "Advanced setup - Claude Code Docs"
[3]: https://docs.anthropic.com/en/docs/claude-code/quickstart?utm_source=chatgpt.com "Quickstart - Claude Code Docs"
[4]: https://docs.anthropic.com/en/docs/claude-code/memory "How Claude remembers your project - Claude Code Docs"
[5]: https://docs.anthropic.com/en/docs/claude-code/best-practices "Best Practices for Claude Code - Claude Code Docs"
[6]: https://docs.n8n.io/advanced-ai/accessing-n8n-mcp-server/ "Accessing n8n MCP server | n8n Docs "
[7]: https://github.com/czlonkowski/n8n-mcp "GitHub - czlonkowski/n8n-mcp: A MCP for Claude Desktop / Claude Code / Windsurf / Cursor to build n8n workflows for you · GitHub"
[8]: https://docs.anthropic.com/en/docs/claude-code/mcp "Connect Claude Code to tools via MCP - Claude Code Docs"
[9]: https://docs.n8n.io/api/ "n8n public REST API Documentation and Guides | n8n Docs "
[10]: https://docs.anthropic.com/en/docs/claude-code/slash-commands "Extend Claude with skills - Claude Code Docs"
[11]: https://docs.n8n.io/workflows/export-import/ "Export and import workflows | n8n Docs "
