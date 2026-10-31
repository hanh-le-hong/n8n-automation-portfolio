# n8n automation portfolio

Marketing and GTM automation built with n8n: lead capture, AI qualification, CRM sync, follow-up, reporting. Every project ships with importable workflow JSON, a node-by-node build guide, design decisions, and what I would change for production.

I come from the demand side of these systems: 8+ years in growth and product marketing (fintech super-app, SaaS, ride-hailing), briefing data and engineering teams for the pipelines I now build myself.

## Projects

| # | Project | Problem | Stack | Status |
|---|---|---|---|---|
| 01 | [AI lead qualification & routing](projects/01-lead-qualify-route/) | Leads from many sources into one CRM, scored by AI, routed by tier, auto follow-up, error alerts. Built from a real Upwork brief. | n8n, Gemini, HubSpot, Sheets, Gmail, Discord | Built, spec project |

## How I work

- **One lead schema** early, so new sources are one node, not a rebuild.
- **Integrations in sub-workflows**, so swapping a CRM is one change in one place.
- **Upsert everywhere.** Re-running a workflow must never duplicate data.
- **Error workflow on every workflow.** Silent failure is the only unacceptable failure.
- **Rules in prompts, not code**, where the client needs to tune them.
- **Count items before clicking.** N in, M out, every node.

## Contact

April Le · [Upwork] · [LinkedIn] · lehhanh74@gmail.com
