# Brief: Lead Capture & CRM Sync

## Goal
When a new Typeform response arrives, create or update a HubSpot contact,
classify the lead quality score, notify Slack for high-intent leads (score > 80),
and log the full record to Airtable.

## Trigger
Typeform webhook (fires on each new form submission)

## Inputs
- `name`: respondent's full name
- `email`: respondent's email address
- `company`: company name
- `role`: job title / role
- `budget`: approximate budget range (text)
- `timeline`: when they need this (text)
- `notes`: freeform notes field

## Outputs
- HubSpot contact created or updated (matched by email)
- Lead score (0–100) assigned to HubSpot contact property
- Slack DM to `#leads` channel if score > 80
- Row added to Airtable "Leads" table

## External Systems
| System | Action | Credentials You Have? |
|---|---|---|
| Typeform | Incoming webhook trigger | Yes (webhook secret) |
| HubSpot | Create/update contact | Yes (Private App token) |
| Slack | Post message to channel | Yes (Bot token) |
| Airtable | Insert row | Yes (Personal Access Token) |

## Constraints
- Do NOT send Slack alert if `email` field is empty or null
- Deduplicate HubSpot contacts by email (use upsert, not create)
- Retry HubSpot calls on 429 rate limit errors
- Lead score logic: budget > $10k AND timeline < 3 months → score 90, etc. (define scoring in workflow)
- Store raw Typeform payload in Airtable for audit trail

## Error Behavior
- HubSpot failure: log to Airtable error table + Slack alert to `#workflow-errors`
- Airtable failure: log to n8n execution log (non-critical, don't alert)
- Slack failure: log to execution (non-critical, continue)

## Edge Cases
- Typeform submission with empty email → skip HubSpot + Slack, still log to Airtable
- HubSpot duplicate email → upsert (update if exists)
- Typeform sends duplicate submission → idempotency check by Typeform response ID
- Budget field is empty → default score to 50

## Sample Input Data
```json
{
  "event_id": "01HXYZ123",
  "form_response": {
    "form_id": "abc123",
    "token": "unique-response-token",
    "submitted_at": "2026-03-06T12:00:00Z",
    "answers": [
      { "field": { "id": "f1", "ref": "name" }, "text": "Jane Smith" },
      { "field": { "id": "f2", "ref": "email" }, "email": "jane@acme.com" },
      { "field": { "id": "f3", "ref": "company" }, "text": "Acme Corp" },
      { "field": { "id": "f4", "ref": "role" }, "text": "Head of Marketing" },
      { "field": { "id": "f5", "ref": "budget" }, "text": "$15,000-$25,000" },
      { "field": { "id": "f6", "ref": "timeline" }, "text": "Within 1 month" },
      { "field": { "id": "f7", "ref": "notes" }, "text": "Very interested, wants a demo ASAP." }
    ]
  }
}
```

## Notes / Context
This is a net-new workflow. No existing workflow to replace.
HubSpot property for lead score is `lead_score` (custom property, already created).
Airtable base ID and table name to be confirmed before deployment.
