# Proposal: AI lead capture, qualification and routing with n8n

Answers to the three questions in the brief. Written to be pasted into an Upwork proposal, then trimmed to the client's tone.

---

## 1. Relevant project: what I built

I built a working version of this exact pipeline as a spec project, using your brief as the spec.

**Flow:** form submit → normalize to one lead schema → duplicate check → Gemini scores 1 to 10 with reason and next action (strict JSON) → saved to a lead table → routed by tier.

- Hot: pushed to CRM as SQL, sales gets a Discord alert with the AI's reasoning and a suggested next step.
- Warm: pushed to CRM as Lead, personalized follow-up email goes out within seconds.
- Cold: pushed to CRM as Subscriber. No noise for the team, no lead lost.

Plus a separate error workflow that posts the failing node and a link to the execution whenever anything breaks.

Repo with workflow files, build guide and design decisions: https://github.com/hanh-le-hong/n8n-automation-portfolio
2-minute walkthrough: [Loom link]

Stack: n8n, Google Gemini, HubSpot, Gmail, Discord. The CRM step is isolated in a sub-workflow, so swapping HubSpot for Pipedrive, Zoho, Close or a Google Sheet is one node change.

## 2. How I would approach your project

**Week 1: Discovery and the first live source.**
Two things first: which CRM you use, and which fields sales actually needs on a lead. Then I connect your website form or landing page via webhook, run it through qualification and into your CRM. Live by end of week 1, with error alerts. This proves the pipeline before touching the harder channels.

**Week 2 to 3: Ads and follow-up.**
Meta or Google Lead Ads via their native n8n triggers. Follow-up sequence of 2 to 3 emails with exit-on-reply, so nobody gets a nurture email after they already answered.

**Week 4+: Messaging channels.**
WhatsApp through Meta Business API or a provider like 360dialog or Twilio. Conversations are unstructured, so I add an AI extraction step (name, company, intent) that feeds the same pipeline. This needs Meta approval, so it goes last.

**Throughout:** every workflow has an error handler, every CRM write is an upsert (re-running never creates duplicates), and you get a one-page handover doc per workflow so your team can change the qualification rules without touching code. The scoring rules live in a plain-text prompt, not JavaScript.

## 3. Which lead source first, and why

**Website forms and landing pages.**

Three reasons:
1. **Highest intent.** Someone who fills in "what are you trying to solve" is further along than an ad click. These are the leads that cost the most to miss.
2. **Structured data.** Form fields map cleanly to CRM fields. No AI extraction needed, so qualification is more accurate on day one.
3. **Fastest to go live.** One webhook. Live in days, which gives us real leads to tune the scoring prompt on before ads add volume.

WhatsApp last, deliberately. It needs Meta API approval and the data is free-text conversation, so it benefits most from a scoring prompt that has already been tuned on cleaner sources.

---

## Questions I would ask before quoting

1. Which CRM, and does it have an API or a native n8n node?
2. Monthly lead volume across all sources, roughly?
3. Who receives hot-lead alerts, and where (Slack, email, WhatsApp)?
4. Do you have existing email templates for follow-up, or should I draft them?
5. n8n Cloud or self-hosted? If you do not have a preference, I will recommend based on volume and data policy.

## Pricing shape (for my own reference, adjust per client)

| Phase | Deliverable | Hours |
|---|---|---|
| 1 | Form → AI → CRM, error alerts, handover doc | 12 to 16 |
| 2 | Lead Ads trigger, 3-step follow-up, exit on reply | 10 to 14 |
| 3 | WhatsApp intake + extraction | 12 to 20 (depends on provider) |
| Ongoing | Monitoring, prompt tuning, new sources | Retainer |
