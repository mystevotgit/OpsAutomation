# AI Production Operations Copilot (OpsAutomation) - n8n Import Pack

This pack includes the full alert-handling flow, the error workflow, a scheduled error generator for demo, and the Jira automation rule used by the feedback loop.

Import the n8n workflows one by one, then import the Jira rule separately in Jira if you want the status-transition callback wired up.

## Files
1. AI Production Operations Copilot - Alert Triage.json
2. AI Production Operations Copilot - Approval Handler.json
3. AI Production Operations Copilot - Error Notification.json
4. AI Production Operations Copilot - Scheduled Alert Generator.json
5. jira-automation-rule-019e7434-f5c7-7d87-9061-5d5ea978cb79-202605292002.json
6. presentation-guide.html

## Workflows

### 1. AI Production Operations Copilot - Alert Triage
Main intake workflow for production incidents.

It:
- receives alert payloads through the `/webhook/ops-copilot-alert` endpoint,
- normalizes the incident,
- calls OpenAI for triage guidance,
- creates or searches Jira issues,
- posts a Slack notification,
- stores incident-learning content in Confluence,
- and emits approval links when the triage recommends rollback or retry.

### 2. AI Production Operations Copilot - Approval Handler
Receives approval decisions from `/webhook/ops-copilot-approval`.

It:
- validates the approval request,
- simulates rollback or retry for approved incidents,
- records rejections when the action is denied,
- and stores approval history in workflow static data for the demo.

### 3. AI Production Operations Copilot - Error Notification
Acts as the error workflow for the demo.

It:
- triggers on workflow errors,
- formats the error details,
- and sends the notification to Slack.

### 4. AI Production Operations Copilot - Scheduled Alert Generator
Generates demo incidents every 15 minutes and posts them to the main alert triage webhook.

It:
- creates rotating sample incident payloads,
- posts them to `/webhook/ops-copilot-alert`,
- and summarizes the response from the main workflow.

After import, update the `targetWebhookUrl` and `approvalBaseUrl` values inside the code node so they point to your n8n domain.

### 5. Jira automation rule export
The Jira rule export is not an n8n workflow, but it supports the closed-loop demo.

It forwards Jira status transitions to `/webhook/jira-status-change` so the main workflow can react to issue updates.

## Webhook paths
- Main alert intake: `/webhook/ops-copilot-alert`
- Approval handler: `/webhook/ops-copilot-approval`
- Jira status feedback: `/webhook/jira-status-change`

For manual testing, n8n also exposes `/webhook-test/...` variants while a workflow is being edited, but the scheduled generator and production setup should use the normal `/webhook/...` endpoints.

## n8n configuration

### Required environment variables
- `OPENAI_API_KEY` - used by the alert triage workflow to call OpenAI.
- `JIRA_BASE_URL` - Jira/Confluence base URL used by the alert triage workflow.
- `JIRA_BASIC_AUTH` - Jira basic-auth credential value used by the Jira HTTP nodes.
- `JIRA_PROJECT_KEY` - Jira project key used when creating issues.
- `CONFLUENCE_SPACE_ID` - Confluence space ID used when storing incident-learning pages.
- `YOUR_N8N_DOMAIN` - your n8n host name, used in the demo payloads and webhook links.

### n8n credentials to configure
- Slack OAuth2 credential for the Error Notification workflow.
- OpenAI credential for the AI triage/OpenAI nodes if you prefer credential-based auth over env vars.
- Jira Cloud credential if you are not using basic auth in the HTTP nodes.

## Import order
1. Import and activate AI Production Operations Copilot - Alert Triage.
2. Import AI Production Operations Copilot - Approval Handler.
3. Import AI Production Operations Copilot - Error Notification and connect it as the error workflow if n8n does not preserve that link automatically.
4. Import AI Production Operations Copilot - Scheduled Alert Generator.
5. Import the Jira automation rule export in Jira and point it at your n8n domain.

## Demo alert payload
POST to `/webhook-test/ops-copilot-alert` or `/webhook/ops-copilot-alert`:

{
  "incidentId": "inc-demo-001",
  "source": "grafana",
  "title": "High latency and error rate on booking-transformer",
  "service": "booking-transformer",
  "environment": "prod",
  "severity": "critical",
  "description": "P95 latency and 5xx rate exceeded threshold after recent deployment",
  "metrics": {
    "p95LatencyMs": 2400,
    "p99LatencyMs": 5100,
    "errorRatePercent": 12.5,
    "throughputRps": 460,
    "saturationPercent": 91
  },
  "logs": [
    "ERROR timeout calling downstream inventory-api after 2000ms",
    "WARN retry attempt=3 correlationId=demo-123",
    "ERROR circuit breaker open for dependency=inventory-api"
  ],
  "approvalBaseUrl": "https://YOUR_N8N_DOMAIN/webhook/ops-copilot-approval"
}

## Simplifications
- Metrics, logs, and runbooks are mocked in code to keep the demo self-contained.
- Rollback and retry are simulated and do not call deployment APIs.
- Incident learning uses workflow static data posted to confluence as knowledgebase. A real vector database can be used for imporvement.
- The scheduled generator is a demo-only helper that posts sample incidents into the main triage flow.

## Presentation deck
Open `presentation-guide.html` in a browser while recording the demo. It provides a colorful horizontal slide deck for the project, a sample payload copy button, and direct presentation prompts for each part of the flow.
