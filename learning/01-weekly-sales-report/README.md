# Learning log 01: Weekly sales report (n8n Quickstart, Level 1)

Exercise from n8n's official QS101 course. Not a client project. Logged here because I rebuilt it to understand the data model, not just to pass the module. Course text is not reproduced; these are my notes.

**Date:** 31 Oct 2026. Zero n8n experience at 12:00, this workflow running end-to-end by 17:00.

## The task

A fictional analytics manager spends every Monday pulling orders from a legacy warehouse, summing the Booked ones for a Discord announcement, and listing the Processing ones for sales to review. Manual, error-prone, already got him in trouble once.

## What I built

```
Manual trigger ─► HTTP Request (GET, header auth) ─► If (orderStatus)
   1 item             30 items                          │
                                          ┌─────────────┴──────────────┐
                                     true: 14 (processing)       false: 16 (booked)
                                          │                            │
                                   Edit Fields (keep 2)          Code: count + sum
                                          │                            │ 1 item
                                   Data Table: Upsert              Discord webhook
                                   (match on orderID)             "16 orders, $2251.14"
```

Screenshots in `screenshots/`. Workflow export: `workflow.json` (replace credential IDs before import).

## Three things I actually learned

**1. n8n runs per item, not per workflow.**
30 orders in = every downstream node runs 30 times. The If node does not create or destroy items, it only sorts them: 14 + 16 = 30. Before adding any node I now ask "N in, M out?" This single habit caught every bug I hit today.

**2. The `json` wrapper and `$('NodeName')`.**
Inside the Code node, data is `items[i].json.orderPrice`, not `items[i].orderPrice`. Return must be `[{ json: {...} }]`. And to reach a node that is not the one directly before: `$('GetDataFromWarehouse').params...` for config, `$('Node').item.json.field` for output. That second form is how you get the original lead back after an AI node has replaced it.

**3. Upsert, never Insert, when a workflow will be re-run.**
Insert on a second run gives 28 rows instead of 14. Upsert on `orderID` keeps 14 and bumps `updatedAt`. Every write in my client-facing work is now an upsert with an explicit match key.

## What I would change for a real client

| Course choice | Production choice | Why |
|---|---|---|
| Manual trigger | Schedule trigger, Monday 08:00 local | The whole point is "every Monday without Nathan" |
| Code node for sum | Summarize node | Zero code, client can edit it |
| n8n Data Table for the Processing list | Google Sheets | Sales managers do not open n8n |
| No error handling | Error workflow → ops channel | If Monday's post is missing, someone must know by 08:05 |
| Hardcoded API key in a credential named "Header Auth account" | Credential named for the system (`ABCorp warehouse API`) | You will have 20 credentials in 3 months |

## Mistakes I made, so I do not repeat them

- Pasted the course's JSON data-structure example into the Code node instead of the JavaScript. Error: "Code doesn't return items properly." JS has `let`, `for`, `return`. JSON is only `"key": value`.
- Looked for an "Add Parameter" button in Send Headers. The 2.x UI pre-creates `Header 1`; you expand it.
- Clicked "Connect to Discord" (OAuth) instead of "setup manually" (webhook URL).
