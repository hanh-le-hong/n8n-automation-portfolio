# AI Lead Qualification & Routing (n8n)

**Spec project.** Built from a real Upwork brief (Oct 2026), not for a paying client. Runs end-to-end on free tiers: n8n Cloud trial, Google Gemini (AI Studio free key), HubSpot Free, Gmail, Discord webhook.

> Brief, in the client's words: *"Leads come from ads, landing pages, forms, WhatsApp and messaging. I want them captured into one CRM, qualified by AI, routed, with automated follow-up. Not building a CRM. Integrate the tools we already use. Reliable, easy to manage, with error handling and docs."*

---

## What it does

A lead submits a form. Within ~5 seconds:

1. Lead is normalized into one standard schema (so every future source looks the same).
2. Duplicates are caught and logged, not re-processed.
3. Gemini scores the lead 1 to 10 with a reason and a suggested next action, returned as strict JSON.
4. Lead + score are saved to an n8n Data Table (audit trail).
5. Lead is routed by tier:
   - **Hot (8 to 10):** pushed to CRM as Sales Qualified Lead, sales gets a Discord alert with the AI reasoning.
   - **Warm (5 to 7):** pushed to CRM as Lead, personalized follow-up email #1 goes out automatically.
   - **Cold (1 to 4):** pushed to CRM as Subscriber. No alert, no email. Nothing lost, nobody interrupted.
6. If anything fails, a separate error workflow posts the failing node, error and execution link to Discord.

## Architecture

```
┌─────────────────────────────── MAIN WORKFLOW ───────────────────────────────┐
│                                                                              │
│  Form ──► Normalize ──► Find existing ──► Is new? ──┬─ no ──► Log duplicate │
│  (1)       (1)            (0 or 1)                  │                        │
│                                                     └─ yes                   │
│                                                        │                     │
│                                   Restore lead ◄───────┘                     │
│                                        │                                     │
│                                   Qualify with AI  ◄── Gemini + JSON parser │
│                                        │ (1, +score/tier/reason)             │
│                                   Merge lead+score                           │
│                                        │                                     │
│                                   Save to Data Table (upsert on email)       │
│                                        │                                     │
│                                   Route by tier                              │
│                              ┌─────────┼──────────┐                          │
│                             hot       warm       cold                        │
│                              │         │          │                          │
│                        ┌─────┴─────┬───┴────┬─────┴─────┐                   │
│                        │  Push to CRM (sub-workflow)     │                   │
│                        └─────┬─────┴───┬────┴───────────┘                   │
│                              │         │                                     │
│                       Discord alert  Gmail follow-up #1                      │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────── SUB-WORKFLOW: Push to CRM ────────────┐   ┌──── ERROR HANDLER ────┐
│ Trigger ──► Which CRM? ──┬─ hubspot ─► HubSpot    │   │ Error trigger         │
│                          ├─ sheets  ─► Sheets     │   │      │                │
│                          └─ other   ─► fail loud  │   │ Discord ops alert     │
└───────────────────────────────────────────────────┘   └───────────────────────┘
```

Item counts are 1 → 1 the whole way (one form submit = one execution). The design still holds for batch sources: a 50-row CSV import would run every node 50 times, except the AI node, which only runs for new leads.

## Design decisions

| Decision | Why | What it costs |
|---|---|---|
| **One canonical lead schema** (`email, full_name, company, website, message, source, submitted_at, lead_id, crm_target`) set right after the trigger | Every new source (ads, WhatsApp, webhook) only needs its own "Normalize" node. Nothing downstream changes. This is what makes "many sources, one CRM" true. | One extra node per source |
| **CRM in a sub-workflow, selected by `crm_target`** | Client said "any CRM we already use." Swapping HubSpot for Pipedrive is one node in one place. Main workflow never knows which CRM it is. | One extra workflow to maintain |
| **Dedupe before AI** | AI calls cost money and time. A returning lead should not be re-scored or re-alerted. | A Data Table lookup, ~50ms |
| **Structured output parser on the AI node** | Forces Gemini to return valid JSON with fixed keys. Without it, one free-text answer breaks the Switch node downstream. | Slightly longer prompt |
| **Rules in the prompt, not in code** (free-email penalty, vendor/student = cold) | Client can tune qualification by editing text, not JavaScript. | Less deterministic than hard rules; mitigated by low temperature (0.2) |
| **Cold leads still go to CRM** | "Fewer leads get missed" was the brief. Cold today can be warm in 6 months. | CRM contact count grows |
| **Upsert everywhere** (Data Table, HubSpot, Sheets) | Re-running a workflow must never create duplicates. | Need a matching key (email) |
| **Error workflow is separate** | One place to see every failure across all workflows. Client asked for error handling explicitly. | 2 nodes |
| **n8n Form Trigger for the demo** | Zero setup, hosted form URL. Production would swap to Webhook (landing page) or native triggers (Meta Lead Ads, Typeform). | Not the client's real form |

## What I would do differently for a paying client

| Phase | Scope | Why this order |
|---|---|---|
| **1 (week 1)** | Website form + landing page via Webhook → this workflow → their CRM. Error alerts. Handover doc. | Highest intent, structured data, live in days. Proves the pipeline before touching messaging channels. |
| **2 (week 2 to 3)** | Meta/Google Lead Ads native trigger. Follow-up sequence #2 and #3 with exit on reply. | Ads leads are high volume, lower intent: AI scoring earns its keep here. |
| **3 (week 4+)** | WhatsApp via Meta Business API or a provider (Twilio, 360dialog). Conversation → AI extracts name/company/intent → same pipeline. | Needs Meta approval and a conversation-to-lead extraction step. Riskiest, so last. |

Also: move from n8n Cloud to self-hosted if volume passes ~2,000 executions/month, add a weekly "leads by tier" summary, and log AI scores vs. actual close rate to recalibrate the prompt.

## Repo contents

```
workflows/
  01-main-lead-qualify-route.json   import into n8n
  02-sub-push-to-crm.json           import into n8n
  03-error-handler.json             import into n8n
docs/
  build-guide.md                    step-by-step, node by node
  proposal-approach.md              answers to the brief's three questions
  test-leads.md                     5 sample leads to submit and the expected tier
screenshots/                        canvas, Discord alert, HubSpot contact, follow-up email
```

## Stack

n8n 2.45 (Cloud) · Google Gemini 2.5 Flash · HubSpot Free · Google Sheets · Gmail · Discord webhook · n8n Data Tables

## Author

April Le, B2B SaaS Product Marketing & GTM, building marketing automation with n8n.  
