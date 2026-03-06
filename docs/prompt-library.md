# Master Prompt Library
## Copy-paste prompts for every phase of workflow building

Use these prompts verbatim (customizing only the `<n>` placeholders).
They are ordered for the standard build pipeline defined in `CLAUDE.md`.

---

## 🔵 PROMPT 0 — Session Start (use at the beginning of every session)

```
Read CLAUDE.md and confirm:
1. Which MCP servers are active (n8n-docs and n8n-runtime)
2. The n8n instance URL you'll be connecting to
3. The skills available in .claude/skills/
4. The current state of workflows/drafts/ (list files)
5. Any open briefs in briefs/ that haven't been built yet

Do not start any workflow work until you've confirmed all of the above.
```

---

## 🔵 PROMPT 1 — New Workflow: Spec Phase

```
Read CLAUDE.md.
Read briefs/<n>.md.
Use the brief-to-workflow skill.

Process in this exact order:
1. Parse the brief — list trigger, inputs, outputs, external systems, credentials, constraints
2. Search existing n8n templates: search_templates("<keywords from brief>")
3. For each template found: state if it's usable, adaptable, or irrelevant and why
4. Identify every node needed: search_nodes("<service>") for each external system
5. For each identified node: get_node("<node-type>") and read docs

Then return ONLY the workflow spec in the Output Contract format from CLAUDE.md.
Do NOT build anything yet.
Flag every open assumption explicitly.
Wait for my review before proceeding.
```

---

## 🔵 PROMPT 2 — New Workflow: Build Phase (after spec is approved)

```
The spec for <workflow name> is approved.
All open assumptions have been resolved as follows:
<paste resolved assumptions here>

Now convert the spec into an n8n implementation.
For each node, provide:
- node name (descriptive)
- node type (exact string from your node research)
- purpose (one sentence)
- key parameters (with values or placeholders)
- input fields consumed
- output fields produced
- error handling approach

Apply the error-handling-patterns skill for every external API call.

Do not generate JSON yet — produce the node-by-node implementation plan first.
Wait for my review.
```

---

## 🔵 PROMPT 3 — New Workflow: Generate JSON

```
The implementation plan is approved.

Now generate the complete n8n workflow JSON.
Before finalizing:
1. validate_node() for every node
2. validate_workflow() on the full JSON
3. If any issues found: run n8n_autofix_workflow() and re-validate

Save the final JSON to workflows/drafts/<n>.json

Also generate:
- payloads/<n>-happy-path.json (with _expected_behavior annotations)
- payloads/<n>-edge-case.json (with _expected_behavior annotations)

Return:
- validation report (blocking issues + non-blocking + go/no-go)
- list of fields that require manual credential binding in n8n
- any remaining manual configuration steps
```

---

## 🔵 PROMPT 4 — Audit Before Deployment

```
Run the workflow-audit skill on workflows/drafts/<n>.json

Check all 7 categories:
1. Naming and readability
2. Architecture quality
3. Error handling
4. Data handling
5. Performance and reliability
6. Credential and security hygiene
7. Maintainability

Return:
- top risks (ordered by severity)
- exact fixes required (with node names)
- recommended improvements
- overall assessment: Production Ready / Needs Work / Redesign
```

---

## 🔵 PROMPT 5 — Deploy

```
Run the validate-before-deploy skill on workflows/drafts/<n>.json
Then run the deployment-checklist skill.

Return:
1. Final validation report (GO / NO-GO)
2. Complete deployment checklist with every manual step I need to do in n8n
3. Test commands I can run to verify the deployment worked
4. Any remaining risks or things to monitor after deployment
```

---

## 🟡 PROMPT 6 — Update Existing Workflow

```
I need to update the existing workflow: <workflow name> (ID: <id if known>)

Change required: <describe the change>

Before making any changes:
1. Export the current workflow: n8n_get_workflow(<id>) → save to workflows/exports/<n>-backup-<date>.json
2. Research any new/changed nodes needed
3. Propose the minimal diff (what changes, what stays the same)
4. Wait for my approval

Then after approval:
5. Apply changes to workflows/drafts/<n>-v<next version>.json
6. Run full validation
7. Run audit
8. Return deployment steps
```

---

## 🟡 PROMPT 7 — Debug Failing Workflow

```
Workflow "<name>" is failing. Here is the error from the execution log:
<paste error>

And the last execution input data:
<paste input data if available>

Diagnose the issue:
1. Identify the failing node
2. Research that node's known issues: get_node("<node-type>")
3. Check the field mapping against the actual input shape
4. Identify the root cause

Return:
- root cause
- exact fix (node name + parameter + correct value)
- whether this is a data issue, configuration issue, or credential issue
- updated test payload that reproduces the bug
- fix to apply to the workflow JSON
```

---

## 🟡 PROMPT 8 — Refactor / Audit Existing Workflow

```
Run the workflow-audit skill on this existing workflow.
Retrieve it with: n8n_get_workflow(<id>)

I want to know:
1. Is this workflow production-ready?
2. What are the top 3 reliability risks?
3. Is there a simpler architecture that achieves the same result?
4. Are there any security issues?

Return the full audit report.
If a redesign is recommended, sketch the new architecture but don't build it until I approve.
```

---

## 🟡 PROMPT 9 — Node Research (standalone)

```
Use the n8n-node-research skill.
Research the following: <service name / use case>

I need to know:
1. The best native n8n node for this
2. All available operations
3. Required vs optional parameters
4. Output shape
5. Any pagination or rate limit gotchas
6. Recommended configuration for <specific use case>

Show me a real template that uses this node if one exists.
```

---

## 🟢 PROMPT 10 — The "One Prompt" Full Build (experienced users)

```
Read CLAUDE.md.
Read briefs/<n>.md.

Execute the complete build pipeline:
1.  search templates
2.  search + read docs for all nodes needed
3.  produce workflow spec in Output Contract format
4.  [PAUSE — show me spec, wait for approval]
5.  implement node-by-node plan with error handling
6.  [PAUSE — show me plan, wait for approval]
7.  generate workflow JSON
8.  validate all nodes
9.  validate full workflow
10. autofix if needed and re-validate
11. run workflow-audit
12. save to workflows/drafts/<n>.json
13. generate test payloads
14. run deployment-checklist skill

Final return:
- spec summary
- validation report
- audit report
- manual steps I must complete in n8n
- test commands
```

---

## Notes on Using These Prompts
- Always use Prompt 0 at the start of a new session
- Always use Prompt 3 before Prompt 5 — never skip validation
- Prompts 1–5 are the standard pipeline for any new workflow
- Prompts 6–9 are for maintenance and debugging
- Prompt 10 is for experienced users who want to move fast — it still enforces the pipeline
