# deployment-checklist

Generate and execute a step-by-step deployment guide for promoting an n8n workflow from draft to production.
Use this skill as the final step before any workflow goes live.

---

## Trigger
Invoked when:
- A workflow has passed `validate-before-deploy` and `workflow-audit`
- User is ready to deploy to their n8n instance
- A workflow is being updated (not just new deploys)

---

## Pre-Deployment Requirements
The following must ALL be true before deployment begins:

- [ ] `validate-before-deploy` completed with GO status
- [ ] `workflow-audit` completed with no blocking issues
- [ ] All open assumptions resolved
- [ ] All required credentials confirmed to exist in n8n instance
- [ ] Test payloads created and field names verified against workflow
- [ ] Backup of any existing workflow with the same name exported

---

## Deployment Steps

### Phase 1 — Backup (if updating existing workflow)
```
n8n_list_workflows()  → find existing workflow ID
n8n_get_workflow(<id>)  → retrieve current JSON
Save to: workflows/exports/<name>-v<n>-backup-<date>.json
```
**Do not proceed without a backup if overwriting an existing workflow.**

---

### Phase 2 — Import / Update

**Option A — Via MCP (recommended if n8n-docs MCP has API access)**
```
n8n_create_workflow(<draft JSON>)      ← for new workflows
n8n_update_full_workflow(<id>, <JSON>) ← for updates
```

**Option B — Via n8n UI (if MCP API access not available)**
1. Open n8n at `http://100.80.13.56:5678`
2. Click `+ New Workflow` or open existing workflow
3. Click `⋮` menu → `Import from File`
4. Select `workflows/drafts/<n>.json`
5. Confirm import

---

### Phase 3 — Manual Configuration in n8n
After import, these steps MUST be done manually in the n8n UI:

- [ ] **Bind credentials**: each credential slot → select the correct saved credential
- [ ] **Configure webhook path**: verify path is correct and not duplicating another workflow
- [ ] **Set schedule**: verify cron expression and timezone if schedule-triggered
- [ ] **Fill in any hardcoded placeholders**: IDs, URLs, field names flagged during build
- [ ] **Fill in workflow description**: use the template from CLAUDE.md

---

### Phase 4 — Test in Dev Mode
1. Open the workflow in n8n editor
2. Click `Test Workflow`
3. Send `payloads/<n>-happy-path.json` via curl or webhook tester:
   ```bash
   curl -X POST http://100.80.13.56:5678/webhook-test/<path> \
     -H "Content-Type: application/json" \
     -d @payloads/<n>-happy-path.json
   ```
4. Verify each node output in the execution view
5. Repeat with `payloads/<n>-edge-case.json`
6. Confirm error branches trigger correctly (send a malformed payload)

---

### Phase 5 — Activate and Publish
1. Fix any issues found in Phase 4
2. Click `Activate` toggle (top right)
3. Confirm workflow status shows `Active`
4. If this workflow should be MCP-accessible:
   - Enable MCP access in workflow settings
   - Confirm it appears in n8n's MCP-eligible workflows list

---

### Phase 6 — Post-Deploy Verification
- [ ] Trigger the workflow with a real or realistic test input
- [ ] Confirm execution completes in the `Executions` tab
- [ ] Confirm outputs appear correctly in destination systems
- [ ] Monitor for first 3 natural executions

---

### Phase 7 — Documentation
- [ ] Export final deployed workflow to `workflows/exports/<name>-v<n>-deployed-<date>.json`
- [ ] Update `docs/deployment-checklist.md` with deploy date and version
- [ ] Commit to git: `git add . && git commit -m "deploy: <workflow name> v<n>"`

---

## Rollback Procedure
If a deployment causes issues:
1. Deactivate the workflow immediately
2. Import the backup JSON from `workflows/exports/`
3. Re-activate
4. Investigate the failure in Executions tab
5. Fix in draft before re-deploying

---

## Output Format

```markdown
## Deployment Checklist: [Workflow Name] v[n]

### Status
[ ] Pre-deployment checks passed
[ ] Backup taken
[ ] Imported to n8n
[ ] Credentials bound
[ ] Test passed (happy path)
[ ] Test passed (edge case)
[ ] Activated
[ ] Post-deploy verification complete

### Manual Steps Required by User
[Numbered list of specific things user must do in n8n UI]

### Known Risks
[Any remaining concerns or things to monitor]

### Deployment Complete
Date: [ISO timestamp]
Version: [v<n>]
Backup: [filename]
```
