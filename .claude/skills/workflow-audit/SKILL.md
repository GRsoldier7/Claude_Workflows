# workflow-audit

Audit an existing n8n workflow for production readiness, maintainability, and reliability.
Use this after building a new workflow AND periodically on existing production workflows.

---

## Trigger
Invoked when:
- A new workflow has passed validation and is ready for final review
- A production workflow is being reviewed for refactoring
- A workflow has had repeated failures and needs a root-cause review

Input: path to workflow JSON OR `n8n_list_workflows` result + workflow ID

---

## Audit Checklist

### Category 1 — Naming and Readability
- [ ] Workflow name follows `[DOMAIN] Action - Target` convention
- [ ] All node names are descriptive and unique
- [ ] Sticky notes present: one at top (overview), one per major section
- [ ] No generic auto-generated node names (`HubSpot1`, `IF1`, etc.)
- [ ] Workflow description is filled in with: purpose + trigger + inputs + outputs

### Category 2 — Architecture Quality
- [ ] Workflow does ONE thing well (not a "mega-workflow" doing 5 unrelated things)
- [ ] Trigger is the only entry point (no manual triggers mixed with webhooks)
- [ ] Data normalized at the top (one `Set` node cleaning incoming data)
- [ ] Logic flows left-to-right, top-to-bottom — no spaghetti connections
- [ ] Sub-workflows used where logic would be repeated across multiple workflows
- [ ] No dead nodes (nodes not connected to anything)

### Category 3 — Error Handling
- [ ] EVERY external API call has an error branch
- [ ] Error branches lead somewhere useful (not just termination)
- [ ] Retry logic present on rate-limited or flaky APIs
- [ ] Dead-letter / error log node captures: workflow name, node name, error message, timestamp, input data
- [ ] Notifications sent for errors that require human intervention

### Category 4 — Data Handling
- [ ] Only required fields passed between nodes (no giant JSON blobs forwarded wholesale)
- [ ] No sensitive data (API keys, PII) in expression logs or Slack messages
- [ ] Consistent field names used throughout (not mixing `email`, `Email`, `emailAddress`)
- [ ] `IF` and `Switch` conditions use explicit type comparisons (not loose equality)

### Category 5 — Performance and Reliability
- [ ] No unbounded loops (always has a limit or break condition)
- [ ] Pagination handled correctly for API endpoints that return paged results
- [ ] Large payloads: streaming or batching used where appropriate
- [ ] Schedule-triggered workflows: idempotent (safe to run twice without double-processing)
- [ ] Webhook workflows: duplicate detection if idempotency key not provided by source

### Category 6 — Credential and Security Hygiene
- [ ] All credentials use named credential slots (no hardcoded secrets)
- [ ] Webhook paths use non-guessable identifiers
- [ ] Webhook authentication enabled (not open to the public internet without auth)
- [ ] No `console.log` or debug nodes left enabled in production

### Category 7 — Maintainability
- [ ] Any hardcoded values that might change (URLs, IDs, thresholds) are at the top in a `Set` or `Config` node
- [ ] Complex expressions have a comment explaining what they do
- [ ] Version tracked: workflow name or description includes version/date of last major change

---

## Output Format

```markdown
## Workflow Audit Report: [Workflow Name]

### Top Risks (ordered by severity)
1. [Risk — Category — Impact — Fix]

### Exact Fixes Required
- [Node name]: [specific change to make]

### Recommended Improvements
- [Suggestion — why it matters — effort estimate: low/med/high]

### Simplified Redesign Recommendation
[Only include if the workflow has fundamental structural problems]
Current: [describe current shape]
Better: [describe simpler architecture]
Why: [reason]

### Overall Assessment
[ ] Production Ready
[ ] Production Ready with Minor Fixes
[ ] Needs Rework Before Production
[ ] Requires Complete Redesign
```

---

## Rules
- Be specific — cite the exact node name and parameter that needs changing
- Prioritize blocking reliability issues over style suggestions
- If a simplified redesign would reduce node count by 30%+ without losing functionality, recommend it
- Never recommend adding nodes "for future use" — keep it minimal
