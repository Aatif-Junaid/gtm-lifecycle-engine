# GTM Content and Lifecycle Engine (n8n)

A self-hosted n8n workflow that turns public market signals into cited, human-reviewed GTM content drafts, then records LinkedIn-ready, email-ready, and CRM-ready outcomes to a lifecycle log. Nothing publishes without a person approving a row in a Google Sheet.

The design write-up, with the canvas walkthrough and the reasoning behind each control, lives at [aatifmulla.me/gtm-systems.html](https://aatifmulla.me/gtm-systems.html).

## What it does

**Track 1, intake to approval queue.** A schedule trigger reads an RSS feed (EIA Today in Energy), keeps three items, normalizes them into a fixed source schema, removes duplicates on `source_url`, and checks that every signal has a named source. Rows that pass are saved to `source_signals` before the model runs. Claude Opus 5, through the Anthropic chat model subnode with a structured output parser, drafts one LinkedIn-ready post per signal from the cited fields only. The draft lands in `approval_queue` with `approval_status` set to Pending. Rows that fail the source check are written to `lifecycle_log` as `source_rejected`.

**Track 2, approval to lifecycle log.** A second schedule trigger reads `approval_queue`, passes only rows where `approval_status` is Approved and `processed_status` is blank, waits two seconds, then fans out: Gmail sends the approved draft to the operator's own inbox, and the LinkedIn node prepares the post. Each branch records its outcome to `lifecycle_log`. A HubSpot engagement node sits on the canvas, shaped to log the email as a CRM activity, with no credential and no connection.

## Nodes

| Track 1 | Type |
| --- | --- |
| Run GTM Engine Demo | Schedule Trigger |
| Read Market Source | RSS Read |
| Limit Demo Intake | Limit (3) |
| Normalize GTM Signal | Set |
| Deduplicate Signals | Remove Duplicates on `source_url` |
| Require Named Source | If |
| Save Source Evidence | Google Sheets, append or update `source_signals` |
| Log Suppressed Signal | Google Sheets, append or update `lifecycle_log` |
| AI Model | Basic LLM Chain, Anthropic Chat Model, Structured Output Parser |
| Draft From Cited Inputs | Code |
| Write To Approval Queue | Google Sheets, append or update `approval_queue` |

| Track 2 | Type |
| --- | --- |
| Scheduled Approval Check | Schedule Trigger |
| Read Approved Drafts | Google Sheets, read `approval_queue` |
| Approval Gate | Filter |
| Wait. | Wait (2 s) |
| Send Approved Email | Gmail |
| Wait | Wait (2 s) |
| Record Email Send Ready | Google Sheets, append or update `lifecycle_log` |
| LinkedIn Draft Ready. | LinkedIn |
| Record LinkedIn Draft Ready | Google Sheets, append or update `lifecycle_log` |
| HubSpot CRM Log Ready | HubSpot engagement, unconnected |

## The Google Sheet

One spreadsheet, three tabs. Create the tabs with these header rows before the first run.

**`source_signals`** (the audit trail)
`source_url`, `source_name`, `published_date`, `company`, `topic`, `signal_type`, `summary`, `source_quality`, `status`, `is_demo_data`, `source_verified`

**`approval_queue`** (the human control point)
`draft_id`, `company`, `signal`, `channel`, `draft_subject`, `draft_body`, `gtm_takeaway`, `why_it_matters`, `source_url`, `source_verified`, `confidence`, `approval_status`, `reviewer_note`, `approved_at`, `processed_status`, `processed_at`

**`lifecycle_log`** (the event trail)
`event_time`, `run_id`, `company`, `channel`, `action_taken`, `journey_stage`, `hubspot_status`, `analytics_event`, `outcome`, `source_url`, `source_verified`, `notes`, `approved_for_distribution`, `distribution_logged`, `crm_log_ready`, plus the source columns carried through

To approve a draft: open `approval_queue`, set `approval_status` to `Approved`, add a `reviewer_note` and `approved_at`. The scheduled track does the rest.

## Import

1. In n8n, create a new workflow, open the menu, choose Import from File, and select `gtm-content-lifecycle-engine.n8n.json`.
2. Replace `YOUR_GOOGLE_SHEET_ID` in the six Google Sheets nodes with your spreadsheet ID.
3. Attach credentials: Google Sheets OAuth2 (six nodes), Gmail OAuth2 (Send Approved Email), Anthropic API key (Anthropic Chat Model). LinkedIn OAuth2 is optional; the node targets `YOUR_LINKEDIN_ORG_ID` until you set one. HubSpot needs nothing because the node is not connected.
4. Set `sendTo` in Send Approved Email to your own address.
5. Run Track 1 once by hand, approve a row in the sheet, then run Track 2.

Both schedule triggers run every 5 minutes, n8n's default minutes interval; the export omits values that equal a default. Keep the workflow inactive between demos and start runs deliberately.

## What was removed from the export

Credential references, webhook IDs, the spreadsheet ID and cached URLs, the operator's email address, the LinkedIn organization ID, the workflow and instance IDs, and tag IDs.

## Differences from the running instance

Two corrections were made in this export and are still to be mirrored in the live workflow: Read Approved Drafts and Approval Gate combine their two checks with AND (the running copy used OR, which let a Pending row through whenever `processed_status` was blank), and an accidental `Require Named Source` column was removed from Normalize GTM Signal and from the sheet schemas.

Known limitation in both copies: nothing writes `processed_status` back after a send, so an approved row is picked up again on the next run until `processed_status` is filled in by hand. A Google Sheets update on `draft_id` after Record Email Send Ready closes that loop.

## License

MIT. See [LICENSE](LICENSE).
