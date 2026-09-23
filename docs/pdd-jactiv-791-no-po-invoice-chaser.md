# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-791-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (UiPath Cartographer, v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-791 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-791 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Replace a manual daily Coupa review with a fully automated weekday run that identifies invoices with no properly linked purchase order and sends a single Slack summary to the AP SME, enforcing the no-PO-no-pay policy consistently.

**Owning department:** Accounts Payable

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Field | Value |
|-------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable - invoice compliance monitoring |
| Short description | Automated weekday run that queries Coupa for draft/new invoices in the past seven days, excludes credit notes, counts those with no properly linked PO, and sends one Slack DM to the SME; sends nothing on a clean day |
| Required roles | Automation (unattended); SME receives output only |
| Trigger and schedule | Time-based: 10:00 Romania time, weekdays only |
| Volume (items per day / peak) | ~194 qualifying invoices observed in one live sample; daily volume [SME REVIEW] |
| Average handling time | Manual: ~daily ad-hoc effort; Automated target: <2 min per run [DEFAULT] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low; main exception is zero-result clean day (send nothing) |
| Input data | Coupa invoice list (status, invoice date, PO linkage, invoice type) |
| Output data | Slack Block Kit DM to SME with qualifying count and filtered Coupa URL; or silence on a clean day |

## 4. To-Be Process (High Level)

The automation runs unattended at 10:00 Romania time on weekdays. It connects to Coupa, retrieves all invoices dated within the past seven days that have status `draft` or `new`, excludes credit notes, and checks each for a properly linked purchase order. A PO number appearing only in the invoice description does not satisfy the linkage requirement. The count of non-compliant invoices is assembled into a Slack Block Kit message and delivered as a direct message to the SME. If the count is zero, no message is sent.

**Steps that disappear from the manual process:**
- Manual filtering of Coupa invoice list by status and date
- Manual identification and exclusion of credit notes
- Manual inspection of PO-linkage field vs. description field
- Manual grouping of results by requester
- Manual composition and sending of Slack message

**Boundaries:** The automation is read-only against Coupa. No PO creation, invoice approval, record modification or requester follow-up tracking is performed.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Trigger: scheduled job fires at 10:00 Romania time (Europe/Bucharest) on a weekday | Scheduler | Run context initialised with `run_date`, `window_start` (run_date − 7 days), `window_end` (run_date) | Weekday check is enforced by the scheduler; no code-level day check required [DEFAULT] |
| 1.2 | Authenticate to Coupa using stored credentials | Coupa | Valid session / API token obtained | Credentials stored in UiPath Orchestrator credential store [DEFAULT]; see BR-10 re read-only scope |
| 2.1 | Query Coupa invoice list: status = `draft` OR `new`; invoice_date >= `window_start`; invoice_date <= `window_end` | Coupa | Page 1 of matching invoice records returned | Access via Coupa API (REST) [SME REVIEW - confirm API vs. UI access] |
| 2.2 | Paginate through all result pages until no more records | Coupa | Complete list of status draft/new invoices in window | [DEFAULT] assume standard Coupa pagination |
| 2.3 | For each invoice record: check `invoice_type` field; if value indicates credit note, skip record (log exclusion) | Coupa | Credit note records removed from working set | Implements BR-03; exclusion count logged for audit |
| 2.4 | For each remaining invoice: inspect PO-linkage field on invoice lines (not the description field) | Coupa | Boolean flag per invoice: PO properly linked = true/false | Implements BR-01, BR-02; PO text in description only → flag = false |
| 2.5 | Filter to invoices where PO properly linked = false | Automation | Qualifying invoice list | These are the records to be reported |
| 2.6 | Count qualifying invoices → `invoice_count` | Automation | Integer ≥ 0 | Implements BR-05 |
| 3.1 | If `invoice_count` = 0: end run without sending any message | Automation | Run completes cleanly; no Slack message sent | Implements BR-07; clean-day termination |
| 3.2 | If `invoice_count` > 0: build filtered Coupa URL using `window_start` and `window_end` as query parameters | Automation | `coupa_url` string: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` | URL format taken verbatim from source §4.3; [SME REVIEW] confirm production vs. test host |
| 3.3 | Compose Slack Block Kit JSON payload with placeholders substituted: `invoice_count`, `coupa_url`, `window_start`, `window_end`, `run_date` | Automation | Complete Block Kit JSON ready to POST | Title: `:receipt: {{invoice_count}} invoices need a purchase order`; body three icon-led lines; primary button "Open the list in Coupa"; footer with calendar/robot/date; exact JSON to be recorded in SDD §4 per source §4.3 |
| 4.1 | POST Slack Block Kit message as DM to SME Slack member ID `WLX9BD8FN` | Slack | HTTP 200; message delivered to SME DM | Implements BR-06; recipient is Irina Capatina; message is a DM, not a channel post |
| 4.2 | Log run result (success, invoice_count, run_date) | Automation / Orchestrator | Run summary record written | [DEFAULT] Orchestrator job log or output argument |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|----|
| Coupa | API (REST) [SME REVIEW] | HTTPS REST calls to coupahost.com | OAuth2 / API key [SME REVIEW] | UiPath Orchestrator Credential Store [DEFAULT] | Read-only scope; test host `uipath-test.coupahost.com` confirmed in source; production host [SME REVIEW] |
| Slack | API (HTTP) | HTTPS POST to Slack API | Bot token | UiPath Orchestrator Credential Store [DEFAULT] | Block Kit message via HTTP Request activity; recipient by Slack member ID WLX9BD8FN; [SME REVIEW] confirm Slack workspace and bot token provisioning |
| UiPath Orchestrator | Scheduler / credential store | Orchestrator API | Robot credential | Managed by platform [DEFAULT] | Hosts schedule trigger and credential assets |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | Invoices without a properly linked purchase order are in scope for the no-PO-no-pay notification. | BR-001 | 2.4 |
| BR-02 | A PO number present only in the invoice description field does not constitute a properly linked PO. | BR-002 | 2.4 |
| BR-03 | Exclude credit notes from the qualifying population. | BR-003 | 2.3 |
| BR-04 | Include only invoices with status `draft` or `new` and invoice date within the past seven calendar days. | BR-004 | 2.1 |
| BR-05 | The notification reports the total count of qualifying invoices; individual invoices are not listed. | BR-005 | 3.3 |
| BR-06 | Send the Slack DM to SME Irina Capatina (Slack ID WLX9BD8FN) with count, policy note, action request, and filtered Coupa link. | BR-006 | 4.1 |
| BR-07 | Send no message when a successful query returns zero qualifying invoices. | BR-007 | 3.1 |
| BR-08 | No retry, fallback or recovery behaviour is implemented; a run that cannot complete is recorded as a failed run. | BR-008 | all |
| BR-09 | A failed run produces no Slack notification; no second message path exists. | BR-009 | all |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion. | BR-010 | all |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|------------------|--------|
| B1 | Credit note encountered | 2.3 | Invoice type field indicates credit note | Exclude record from working set; log exclusion count; continue processing |
| B2 | Description-only PO | 2.4 | PO text present in description but PO linkage field on lines is empty/invalid | Treat as missing PO; include in qualifying count if other rules pass |
| B3 | Clean day (zero results) | 3.1 | Successful query returns zero qualifying invoices after all filters applied | End run without sending Slack message; log clean run result |
| B4 | Unhandled business exception | any | Any unexpected record state not covered by B1–B3 | Log record details; continue to next record; do not abort run [DEFAULT] |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|------------------|----------|-------------|--------|
| S1 | Coupa authentication failure | Invalid or expired credentials at step 1.2 | High | No retry (BR-08) | Fail run; log error; no Slack message sent (BR-09) |
| S2 | Coupa API unavailable | HTTP 5xx or connection timeout at steps 2.1–2.2 | High | No retry (BR-08) [DEFAULT] | Fail run; log error; report failed run to Orchestrator |
| S3 | Coupa data element not found | Expected field (status, invoice_date, PO linkage) absent from record | Medium | Skip record; log warning [DEFAULT] | Continue processing remaining records |
| S4 | Slack API error | Non-200 response from Slack POST at step 4.1 | High | No retry (BR-08) | Fail run; log error; report failed run to Orchestrator |
| S5 | Network timeout | General network timeout during any HTTP call | High | No retry (BR-08) [DEFAULT] | Fail run; log error |
| S6 | Credential expiry | Orchestrator asset unavailable or credential rotation failure | High | No retry [DEFAULT] | Fail run; alert Orchestrator admin [DEFAULT] |
| S7 | Unhandled exception | Any .NET/UiPath exception not caught by S1–S6 | High | No retry [DEFAULT] | Log full exception; fail run gracefully via Global Exception Handler |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Coupa access method.** [SME REVIEW] Confirm whether the robot uses the Coupa REST API or UI automation; source references read access but does not specify the interface type.
2. **OQ-02 - Production Coupa hostname.** [SME REVIEW] Source shows `uipath-test.coupahost.com`; confirm the production host before go-live.
3. **OQ-03 - Coupa API authentication.** [SME REVIEW] Confirm OAuth2 client credentials vs. API key; required to provision the Orchestrator credential asset.
4. **OQ-04 - Slack bot token provisioning.** [SME REVIEW] Confirm Slack workspace, bot app name, and who provisions and owns the bot token.
5. **OQ-05 - Clean-day message wording.** [SME REVIEW] Source §7.1 says "send a Slack message saying congrats that all invoices have a PO assigned" but BR-07 and §4.2 say "send nothing"; SME must resolve this contradiction — current PDD follows BR-07 (send nothing).
6. **OQ-06 - Reminder wording sign-off.** [SME REVIEW] Source next-steps item 3 requests agreement on final reminder wording and date/amount format; Block Kit JSON must be confirmed before build.
7. **OQ-07 - Coupa PO-linkage field name.** [SME REVIEW] The exact API field name on invoice lines that represents a properly linked PO must be confirmed with Coupa admin.
8. **Scheduler timezone.** [DEFAULT] Europe/Bucharest used for 10:00 Romania time; Orchestrator trigger configured accordingly.
9. **Orchestrator credential store.** [DEFAULT] Coupa and Slack credentials stored as Orchestrator assets; no secrets in code or config files.
10. **Appendix A diagrams.** The source references a current-state process map (§3) and future-state map (§4.1) as embedded images; image content was not read — if diagram content differs from text, SME must flag the discrepancy.

## 11. Success Criteria

1. A weekday test run executes at 10:00 Romania time (Europe/Bucharest) and completes without error.
2. Only invoices with status `draft` or `new` and invoice_date within the past seven days are evaluated; all others are excluded.
3. Credit notes are excluded from the qualifying count and the exclusion is logged.
4. An invoice with a PO number in the description only (no line-level PO linkage) is counted as missing a PO.
5. A Slack Block Kit DM is delivered to Slack member ID WLX9BD8FN containing the correct qualifying count and a working filtered Coupa URL.
6. A successful run returning zero qualifying invoices produces no Slack message and is recorded as a clean run.
7. A run that fails at any step produces no Slack notification and is reported as a failed run in Orchestrator.
8. No Coupa record is created or modified during any run.
