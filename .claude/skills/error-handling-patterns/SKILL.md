# error-handling-patterns

Apply production-grade error handling to any n8n workflow.
Use this skill to add or audit error handling on any workflow that touches external systems.

---

## Trigger
Invoked when:
- Adding error handling to a new workflow during build phase
- Upgrading error handling on an existing workflow
- A workflow has had unexplained failures in production

---

## Standard Error Handling Architecture

Every workflow that calls external APIs must implement this pattern:

```
[External API Node]
   ├─ Success ──────────────────────────────→ [Next Step]
   └─ Error → [Classify Error]
                ├─ Retryable (5xx, timeout, rate limit)
                │     └─ [Wait + Retry] → [External API Node]  (max 3x, exponential backoff)
                │           └─ Still failing → [Dead Letter Log] + [Alert]
                └─ Non-Retryable (4xx, auth, bad input)
                      └─ [Dead Letter Log] + [Alert if critical]
```

---

## Pattern Library

### Pattern A — Simple Retry (for non-critical, idempotent operations)
```
Node Settings:
  - continueOnFail: true
  - retryOnFail: true
  - maxTries: 3
  - waitBetweenTries: 1000ms (with note to increase for rate-limited APIs)
```
Use when: the operation is safe to retry, failure is non-critical, no downstream notification needed.

---

### Pattern B — Error Branch with Notification (for critical operations)
Node structure:
1. **[API Node]** — `continueOnFail: true`
2. **Check for Error** — `IF` node: `{{ $json.error !== undefined }}`
   - TRUE → error path
   - FALSE → success path
3. **Format Error Log** — `Set` node capturing:
   - `workflowName`: `{{ $workflow.name }}`
   - `nodeName`: `[API Node Name]`
   - `errorMessage`: `{{ $json.error.message }}`
   - `errorCode`: `{{ $json.error.httpCode }}`
   - `timestamp`: `{{ new Date().toISOString() }}`
   - `inputData`: `{{ JSON.stringify($input.first().json) }}`
4. **Log to Dead Letter** — Airtable/Postgres/Notion node saving the error record
5. **Alert** — Slack/Email node with sanitized error summary (no raw API responses)

---

### Pattern C — Global Error Workflow (for mature setups)
Create a dedicated error-handling workflow:
- Trigger: `Error Trigger` node
- Receives: workflow name, node name, error message, execution ID
- Actions: log to DB + alert Slack + optionally re-queue

Attach to individual workflows via: `Settings > Error Workflow`

This is the recommended pattern once you have more than 5 production workflows.

---

### Pattern D — Rate Limit Handling (for APIs with strict limits)
```
[API Node] → [Check Response Code] → IF 429:
  → [Wait Node: 60 seconds]
  → [Retry API Node]
  → IF still 429: Dead Letter + Alert
```
APIs that need this: HubSpot, Airtable, Salesforce, Twitter, most marketing platforms.

---

### Pattern E — Idempotency Guard (for webhook-triggered workflows)
Prevents duplicate processing when a webhook fires more than once:
```
[Webhook] → [Lookup Execution ID in DB] → IF already processed:
  → [Stop and Return 200 OK]  ← (acknowledge but do nothing)
IF not processed:
  → [Mark as In-Progress in DB]
  → [Main Workflow Logic]
  → [Mark as Completed in DB]
```

---

## Dead Letter Log Schema
Every error log entry must capture:

```json
{
  "workflowName": "string",
  "workflowId": "string",
  "executionId": "string",
  "nodeName": "string",
  "nodeType": "string",
  "errorMessage": "string",
  "errorCode": "number | null",
  "timestamp": "ISO 8601",
  "inputData": "string (JSON stringified, sensitive fields redacted)",
  "retryCount": "number",
  "resolved": false
}
```

---

## Alert Message Template (Slack)
```
🔴 *Workflow Error* — [Workflow Name]
Node: [Node Name]
Error: [Error message — sanitized]
Time: [timestamp]
Execution: [n8n execution URL]
Action needed: [Yes/No + brief description]
```

---

## Rules
- **Every external API call** must have at minimum Pattern A (retry)
- **Critical operations** (payment, data writes, customer-facing) must use Pattern B or C
- **Never pass raw API error responses** to Slack or external logging without sanitizing
- **Dead letter log is mandatory** for any workflow processing data that can't be re-fetched
- **Idempotency guard required** on all webhook-triggered workflows processing financial or CRM data
- Pattern C (Global Error Workflow) is strongly recommended once you reach 5+ production workflows
