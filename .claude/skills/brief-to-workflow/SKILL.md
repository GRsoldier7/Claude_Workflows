# brief-to-workflow

Convert a plain-English automation brief into a complete, validated n8n workflow spec.
This is always the first skill invoked at the start of a workflow build session.

---

## Trigger
Invoked when user provides a `briefs/<n>.md` file or pastes a brief description.

---

## Process

### Phase 1 — Parse the Brief
Extract and structure all of the following. If any are missing, flag them as open assumptions before continuing.

| Field | Description |
|---|---|
| **Goal** | What business outcome does this workflow achieve? |
| **Trigger** | What starts this workflow? (webhook, schedule, manual, event) |
| **Inputs** | What data enters the workflow? (fields, sources, types) |
| **Outputs** | What gets created, updated, or sent? |
| **External Systems** | Which APIs, services, or databases are touched? |
| **Credentials Needed** | For each external system: name, auth type, required scopes |
| **Constraints** | Rate limits, deduplication rules, conditional logic, SLA |
| **Error Behavior** | What should happen when each step fails? |

---

### Phase 2 — Template Search (ALWAYS run this first)
Before designing anything:

```
search_templates("<goal keywords>")
search_templates("<trigger type> + <main service>")
```

For each template found:
- Evaluate if it covers 50%+ of the brief
- Note which nodes it uses (learn from this)
- Note what is missing vs the brief

Recommendation: use, adapt, or discard with reason.

---

### Phase 3 — Node Research
For every external system or transformation needed:

```
search_nodes("<service name>")          → find candidate nodes
get_node("<node type>")                 → read full docs + params
```

Do this for EVERY node before including it in the spec.
Never assume a node's parameters, operations, or field names from memory.

---

### Phase 4 — Produce the Workflow Spec
Return a complete structured spec in this format:

```markdown
## Workflow Spec: [Name]

### Goal
[One-sentence objective]

### Trigger
- Type: [Webhook | Schedule | Manual | Event]
- Source: [Service name + configuration]
- Path/Pattern: [webhook path or cron expression]

### Required Credentials
| Credential Name | Service | Auth Type | Required Scopes |
|---|---|---|---|
| ... | ... | ... | ... |

### Node Plan (ordered)
1. **[Node Name]** — `[node-type]`
   - Purpose: [what it does]
   - Inputs: [fields consumed]
   - Outputs: [fields produced]
   - Key Parameters: [critical config]
   - Error Path: [what happens on failure]

2. [repeat for each node...]

### Branching Logic
[Describe every conditional path, including the "do nothing" path]

### Error Handling Summary
[One line per external call: retry strategy + fallback]

### Edge Cases
[Numbered list of known failure scenarios and how they are handled]

### Open Assumptions
[Anything not confirmed — MUST be resolved before building]

### Test Payloads Required
- `payloads/<name>-happy-path.json` — [description]
- `payloads/<name>-edge-case.json` — [description]

### Deployment Steps
[Manual steps the user must do in n8n after import]
```

---

## Rules
- Prefer the **smallest viable workflow** — never add nodes for "future use"
- Prefer **native n8n nodes** over `HTTP Request` — only use HTTP Request when no native node exists
- **Never invent** credential names, field names, IDs, or paths — flag as open assumption instead
- **Flag every assumption** explicitly in the Open Assumptions section
- **Do not build** until the spec is reviewed and open assumptions are resolved
- If a template covers the use case, adapt it rather than starting from scratch
