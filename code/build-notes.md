# Build Notes — JACTIV-791 No-PO Invoice Chaser

## Plan
1. Read skills (uipath-api-workflow, uipath-platform, uipath-solution)
2. Create solution: `uip solution init no-po-invoice-chaser-791`
3. Create project: `uip api-workflow init no-po-invoice-chaser-api` (inside solution)
4. Extract reference Workflow.json from §4.5 via awk
5. Apply SDD deviation: credit-note filter uses `is-credit-note` field, not `invoice-type`
6. Write bindings_v2.json (Coupa + Slack connections)
7. Write connection resource files under resources/solution_folder/connection/
8. Validate, run gate, pack

## Summary
Single API Workflow project inside a solution; queries Coupa for no-PO invoices in a 7-day window and sends a Slack Block Kit DM when count > 0.

## Task Table

| Task | Project | Status | Notes |
|---|---|---|---|
| Solution scaffold | no-po-invoice-chaser-791 | done | |
| API Workflow project | no-po-invoice-chaser-api | done | init auto-registered in .uipx |
| Workflow.json | no-po-invoice-chaser-api | done | from reference; 1 field changed |
| bindings_v2.json | no-po-invoice-chaser-api | done | Coupa + Slack entries |
| Connection resources | resources/solution_folder/ | done | 2 files |
| validate-build.sh | — | done | passed, 19 activities |
| uip solution pack | — | done | no-po-invoice-chaser-791_0.0.1.zip |

## Deviations from the SDD

- Credit-note exclusion field changed from `inv['invoice-type'] === 'Credit Note'` (reference) to `inv['is-credit-note'] === true` (SDD §4 step 4: "Exclude credit notes (`is-credit-note`)" and test B1: "`is-credit-note = true`").

## Left for a human

None. No selectors or credential values in workflow. OQ-02 (production Coupa hostname) and OQ-03 (auth method) are architect review items resolved at deploy time by re-provisioning the connection.

## How to test this

```bash
# Static validate
uip api-workflow validate code/no-po-invoice-chaser-791/no-po-invoice-chaser-api/Workflow.json --output json

# Runnability gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# Pack
uip solution pack code/no-po-invoice-chaser-791 /tmp/buildcheck \
  --name no-po-invoice-chaser-791 --version 0.0.1 --output json
```
