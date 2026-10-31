# Test leads

Submit these through the form, in order. Expected tier is a prediction; Gemini may land one tier off. If it is two tiers off, tune the prompt.

| # | Full name | Work email | Company | Website | Message | Expected | Checks |
|---|---|---|---|---|---|---|---|
| 1 | Anna Berg | anna@acmesaas.io | Acme SaaS | acmesaas.io | We get about 300 inbound leads a month from our site and ads. Sales can't keep up and we're losing deals. Need qualification and routing into HubSpot. | **hot** | Discord alert, HubSpot contact stage = SQL |
| 2 | Tom Reyes | tom@brightpath.co | BrightPath | brightpath.co | Looking at options for lead management. Early stage, want to understand pricing. | **warm** | HubSpot stage = Lead, Gmail follow-up sent |
| 3 | Minh Nguyen | minh.student@gmail.com | University project | | I'm a student researching CRM automation for my thesis. Can you share materials? | **cold** | HubSpot stage = Subscriber, no alert, no email |
| 4 | Sarah Kim | sarah@growthagency.com | Growth Agency | growthagency.com | We're an agency and would love to partner with you to resell your product to our clients. | **cold** | Same as 3 |
| 5 | Anna Berg | ANNA@AcmeSaaS.io | Acme SaaS | acmesaas.io | Following up on my earlier message. | **duplicate** | Goes to Log duplicate. AI node must NOT run. Email normalized to lowercase matches lead 1. |

After all five: Data Table `leads` has **4 rows** (not 5). HubSpot has 4 contacts. Discord has 1 hot alert and 1 duplicate log. Gmail sent 1 email.

## Error test

Edit the Gemini credential to a wrong key. Submit lead 1 again with a new email. Expected: main workflow fails at "Qualify with AI", error handler posts to `#ops-alerts` with node name and execution link. Restore the key.
