# n8n-node-research

Deep-dive research on a specific n8n node type before including it in a workflow.
Use this whenever you are unfamiliar with a node, when a node has multiple versions,
or when you need to confirm available operations, field names, and pagination behavior.

---

## Trigger
Invoked when:
- A node is needed and its exact parameters/operations are unclear
- A node has recently had a major version change
- A workflow is failing and the root cause might be node misconfiguration

Input: service name OR node type string

---

## Research Process

### Step 1 — Discover
```
search_nodes("<service name>")
```
- Note all matching node types returned
- Identify the canonical/recommended node (usually the newest version)
- Note any deprecated versions to avoid

### Step 2 — Read Full Docs
```
get_node("<node-type>")
```
For each candidate node, capture:

| Field | What to look for |
|---|---|
| **Node Type** | Exact string (e.g., `n8n-nodes-base.hubspot`) |
| **Version** | Current latest stable |
| **Auth Methods** | API Key / OAuth2 / Basic — which is recommended |
| **Resources** | Top-level resource categories (Contacts, Deals, etc.) |
| **Operations** | Available operations per resource |
| **Required Fields** | Which parameters are always required |
| **Optional Fields** | Useful optional parameters |
| **Output Shape** | What the node returns — field names and types |
| **Pagination** | How the node handles paged results |
| **Rate Limits** | Known rate limit behavior / retry support |
| **Gotchas** | Known issues, deprecated fields, version differences |

### Step 3 — Find Examples
```
search_templates("<service name> + <operation>")
```
- Find real-world usage examples
- Note how experienced workflow builders structure this node
- Extract any non-obvious parameter combinations

### Step 4 — Cross-Reference
If the node has both a native version AND an HTTP Request fallback option:
- List pros/cons of each
- Recommend the native node unless there is a specific reason not to

---

## Output Format

```markdown
## Node Research: [Service Name]

### Recommended Node
- Type: `n8n-nodes-base.<name>`
- Version: [x.x]
- Auth: [recommended auth method]

### Available Operations
| Resource | Operations |
|---|---|
| [Resource] | [op1, op2, op3] |

### Key Parameters for [Most Common Use Case]
```json
{
  "resource": "...",
  "operation": "...",
  "requiredField": "...",
  "optionalField": "..."
}
```

### Output Shape
```json
{
  "fieldName": "type — description"
}
```

### Pagination Notes
[How to handle paged results]

### Rate Limit Notes
[Known limits + recommended retry strategy]

### Gotchas / Watch Out For
- [Issue 1]
- [Issue 2]

### Templates Found
- [Template name + what it shows]
```

---

## Rules
- Always use `get_node` — never rely on memory for node parameter names
- If two versions of a node exist, explicitly state which to use and why
- Note any fields that are required vs optional — this prevents validation failures
- If pagination is not handled by the node natively, flag this as a required manual step
