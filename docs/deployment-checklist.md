# Deployment SOP — n8n Workflows

This document defines the standard operating procedure for deploying any workflow
from this repo to the n8n instance at `http://100.80.13.56:5678`.

---

## Golden Rules

1. **Dev first, always.** Never deploy an untested workflow directly to an active schedule or webhook.
2. **Backup before overwrite.** Export the existing workflow before any update.
3. **Validate before deploy.** The `validate-before-deploy` skill must return GO.
4. **Audit before activate.** The `workflow-audit` skill must return no blocking issues.
5. **Never delete.** Archive workflows by deactivating them — do not delete until they've been inactive for 30 days.

---

## Deployment Log

Track every deployment here.

| Date | Workflow Name | Version | Type | Deployed By | Notes |
|---|---|---|---|---|---|
| 2026-03-06 | _(none yet)_ | v1 | New | Aaron | Initial setup |

---

## Rollback Registry

If you ever need to roll back, use this table to find the backup file.

| Workflow Name | Backup File | Date | Reason |
|---|---|---|---|
| _(none yet)_ | — | — | — |

---

## Credential Registry

Track which credentials exist in n8n and what they are used for.
(Do not store values here — this is just a name/purpose registry.)

| Credential Name in n8n | Service | Type | Used In |
|---|---|---|---|
| _(to be filled in)_ | | | |

---

## Active Workflow Registry

| Workflow Name | ID | Status | Trigger | MCP Exposed | Last Updated |
|---|---|---|---|---|---|
| _(to be filled in)_ | | | | | |

---

## Environment Notes

- **n8n URL**: `http://100.80.13.56:5678` (Tailscale — local network only)
- **n8n Version**: check `Settings > About n8n`
- **API Docs**: `http://100.80.13.56:5678/api/v1/docs`
- **Executions**: `http://100.80.13.56:5678/executions`
