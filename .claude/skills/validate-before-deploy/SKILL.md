# validate-before-deploy

Run a complete validation pass on an n8n workflow before it is deployed to any environment.
This skill must be run before EVERY import, update, or deployment — no exceptions.

---

## Trigger
Invoked after a workflow draft has been built and before any deployment action.
Input: path to workflow JSON in `workflows/drafts/<n>.json`

---

## Validation Pipeline

### Step 1 — Node-Level Validation
For every node in the workflow:

```
validate_node("<node-type>", <parameters>)
```

Check for:
- [ ] Node type exists in current n8n version
- [ ] All required parameters are present
- [ ] No deprecated fields used
- [ ] Operation/resource combination is valid
- [ ] Field references use correct expression syntax (`{{ $json.field }}`)
- [ ] Credential reference is present (not hardcoded)
- [ ] Node name is descriptive (not `HTTP Request1` style)

---

### Step 2 — Workflow-Level Validation

```
validate_workflow(<workflow JSON>)
```

Check for:
- [ ] Trigger node is present and configured
- [ ] All nodes are connected (no orphaned nodes)
- [ ] No circular connections
- [ ] Error branches exist for every external API node
- [ ] `Set` nodes used to normalize data before branching
- [ ] Workflow has a name (not "My workflow")
- [ ] Workflow has a description filled in
- [ ] All credential slots are named (not empty)

---

### Step 3 — Integration Checks (manual review)
These cannot be auto-validated and require human confirmation:

- [ ] **Credential names match** what exists in the n8n instance
- [ ] **Field mappings** match actual payload shape (check against test payload)
- [ ] **Rate limits** accounted for (retry/backoff present on rate-limited APIs)
- [ ] **Pagination** handled if API returns paged results
- [ ] **Webhook path** is non-guessable and not reusing another workflow's path
- [ ] **Schedule expression** correct and tested (use cron validator)
- [ ] **Data volume** — no unbounded loops without a limit/break condition

---

### Step 4 — Security Checks
- [ ] No hardcoded API keys, tokens, or passwords in any node parameter
- [ ] No sensitive data passed to logging or notification nodes in plain text
- [ ] Webhook authentication is enabled (header auth or basic auth)
- [ ] Error messages don't leak raw API responses to external services

---

### Step 5 — Test Payload Verification
- [ ] `payloads/<n>-happy-path.json` exists and covers the main flow
- [ ] `payloads/<n>-edge-case.json` exists and covers at least one failure scenario
- [ ] Payload field names match the field names used in the workflow's first node

---

## Output Format

```markdown
## Validation Report: [Workflow Name]

### Blocking Issues (must fix before deploy)
- [Issue description + node name + fix instruction]

### Non-Blocking Improvements (recommended)
- [Issue description + suggested fix]

### Missing Credentials (must be created in n8n)
- [Credential name + type + required scopes]

### Missing or Incomplete Test Payloads
- [What is missing + what field to add]

### Go / No-Go
[ ] GO — no blocking issues found
[ ] NO-GO — [number] blocking issues must be resolved first
```

---

## Rules
- **Never skip** node-level validation even for "simple" nodes
- **Never skip** the workflow-level validation
- **A single blocking issue = NO-GO** — do not deploy until resolved
- If `n8n_autofix_workflow` is available and there are fixable issues, run it and re-validate
- After autofix, always re-run full validation to confirm no regressions
