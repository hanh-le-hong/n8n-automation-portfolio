# Build guide: AI Lead Qualification & Routing

Two ways to build. **Import** (15 min, then wire credentials) or **from scratch** (90 min, you learn every node). Do import first, then rebuild from scratch once without looking.

Time estimate assumes you already have: n8n Cloud, a Gemini API key, HubSpot Free account, a Gmail account, a Discord server where you can create a webhook.

---

## Part A: Prerequisites (20 min)

### A1. Gemini API key
1. Go to aistudio.google.com → **Get API key** → **Create API key**.
2. Copy it. You will paste it into n8n in B3.

### A2. HubSpot Free + Private App token
1. Sign up at hubspot.com (Free CRM).
2. Settings (gear) → **Integrations → Private Apps → Create private app**.
3. Name: `n8n`. Scopes tab: enable `crm.objects.contacts.read` and `crm.objects.contacts.write`.
4. Create → copy the **access token**.

### A3. Discord webhook
1. In your Discord server: channel settings (gear) → **Integrations → Webhooks → New Webhook**.
2. Name it `n8n leads`, pick the channel, **Copy Webhook URL**.
3. Make a second webhook in an `#ops-alerts` channel for the error handler.

### A4. Google Sheet (optional second CRM target)
1. Create a sheet named `Leads CRM`. Row 1 headers, exactly:
   `email | full_name | company | website | message | source | submitted_at | lead_id | crm_target | score | tier | icp_fit | reason | next_action`

### A5. n8n Data Table
1. n8n → **Personal → Data Tables → Create**. Name: `leads`.
2. Columns:

| Column | Type |
|---|---|
| email | String |
| full_name | String |
| company | String |
| website | String |
| message | String |
| source | String |
| submitted_at | String |
| lead_id | String |
| crm_target | String |
| score | Number |
| tier | String |
| icp_fit | Boolean |
| reason | String |
| next_action | String |

---

## Part B: Import and wire (15 min)

### B1. Import three workflows
For each file in `workflows/`: n8n → **Create workflow** → menu `...` (top right) → **Import from file**.

Import in this order:
1. `03-error-handler.json`
2. `02-sub-push-to-crm.json`
3. `01-main-lead-qualify-route.json`

### B2. Credentials (one time)
After import, nodes with a red warning need a credential. Click each node, pick **Create new credential**:

| Node | Credential type | What to paste |
|---|---|---|
| Gemini model | Google Gemini (PaLM) API | key from A1 |
| HubSpot: upsert contact | HubSpot App Token | token from A2 |
| Alert sales (Discord), Log duplicate (Discord) | Discord Webhook | URL from A3 (leads channel) |
| Alert ops (Discord) in error handler | Discord Webhook | URL from A3 (ops channel) |
| Follow-up email #1 (Gmail) | Gmail OAuth2 | click Sign in with Google |
| Google Sheets: upsert row | Google Sheets OAuth2 | Sign in with Google, then pick the sheet |

### B3. Re-select instance-specific things
These cannot travel in a JSON file because their IDs are unique to your n8n:

| Node | What to select |
|---|---|
| Find existing lead | Data table → `leads` |
| Save lead | Data table → `leads` |
| Push to CRM (hot/warm/cold) × 3 | Workflow → `02 - Push to CRM (sub-workflow)` |
| Google Sheets: upsert row | Document → `Leads CRM`, Sheet → `Sheet1` |

### B4. Attach the error handler
Main workflow → `...` → **Settings → Error workflow** → choose `00 - Error handler (global)`. Do the same for the sub-workflow.

### B5. Publish the sub-workflow and error handler
Both must be **Published** (toggle top right) or the main workflow cannot call them.

### B6. Test
1. Open main workflow → click **Lead form** → **Execute step** → **Open test form**.
2. Submit a lead from `docs/test-leads.md`.
3. Watch the canvas. Every node should turn green. Check:
   - Data Table `leads` has 1 row with score and tier.
   - HubSpot has the contact with the right lifecycle stage.
   - Discord got an alert (hot) or Gmail sent an email (warm).
4. Submit the **same email again** → should go to `Log duplicate`, AI node must NOT run.
5. Temporarily break something (wrong Gemini key) → submit → error handler posts to `#ops-alerts`.

---

## Part C: Build from scratch, node by node (90 min)

Build the **sub-workflow and error handler first**, then the main. Main needs to reference them.

### C1. Error handler (5 min)
1. New workflow: `00 - Error handler (global)`.
2. Add first step → search `error trigger` → **Error Trigger**.
3. `+` → **Discord → Send a message**. Connection: Webhook. Credential: ops webhook. Message (Expression):
   ```
   🚨 WORKFLOW FAILED
   Workflow: {{ $json.workflow.name }}
   Node: {{ $json.execution.lastNodeExecuted }}
   Error: {{ $json.execution.error.message }}
   Execution: {{ $json.execution.url }}
   ```
4. Publish.

What you learn: the Error Trigger receives one item with `workflow` and `execution` objects. Every error in every workflow that points here lands in one channel.

### C2. Sub-workflow: Push to CRM (20 min)
1. New workflow: `02 - Push to CRM (sub-workflow)`.
2. Add first step → **When Executed by Another Workflow**. Input data mode: **Accept all data**.
3. `+` → **Switch**. Mode: Rules. Two rules:
   - `{{ $json.crm_target }}` equals `hubspot` → rename output `hubspot`
   - `{{ $json.crm_target }}` equals `sheets` → rename output `sheets`
   - Options → **Fallback output: Extra output** (so unknown values do not silently vanish).
4. From `hubspot` output → **HubSpot → Contact → Create or Update**. Email: `{{ $json.email }}`. Additional fields:
   - First Name: `{{ $json.full_name.split(' ')[0] }}`
   - Last Name: `{{ $json.full_name.split(' ').slice(1).join(' ') }}`
   - Company Name: `{{ $json.company }}`
   - Website: `{{ $json.website }}`
   - Lifecycle Stage: `{{ $json.tier === 'hot' ? 'salesqualifiedlead' : ($json.tier === 'warm' ? 'lead' : 'subscriber') }}`
   - Message: `{{ $json.message }} [AI {{ $json.score }}/10 {{ $json.tier }}] {{ $json.reason }}`
5. From `sheets` output → **Google Sheets → Append or Update Row**. Pick document and sheet. Mapping: Map Automatically. Column to match on: `email`.
6. From `Fallback` output → **Code**. Paste:
   ```javascript
   const t = $input.first().json.crm_target;
   throw new Error(`Unknown crm_target: ${t}. Add a branch in 'Which CRM?'.`);
   ```
   This is on purpose. Silent failures are how leads get lost.
7. HubSpot and Sheets both → **Edit Fields** `Return status`: `crm_status = pushed`, `email = {{ $('When Executed by Another Workflow').first().json.email }}`.
8. Settings → Error workflow → `00 - Error handler`. Publish.

What you learn: a sub-workflow is a function. Input in, output out. The main workflow does not care what happens inside.

### C3. Main workflow (60 min)

**Node 1. Lead form** (n8n Form Trigger)
Title `Request a demo`. Fields: Full name (text, required), Work email (email, required), Company (text, required), Website (text), What are you trying to solve? (textarea, required).
Execute step → Open test form → submit one lead → confirm 1 item out.

**Node 2. Normalize lead** (Edit Fields). Include Other Input Fields: OFF. Nine fields:

| Name | Value (Expression) |
|---|---|
| email | `{{ $json['Work email'].toLowerCase().trim() }}` |
| full_name | `{{ $json['Full name'].trim() }}` |
| company | `{{ $json['Company'] }}` |
| website | `{{ $json['Website'] \|\| '' }}` |
| message | `{{ $json['What are you trying to solve?'] }}` |
| source | `web_form` (Fixed) |
| submitted_at | `{{ $now.toISO() }}` |
| lead_id | `{{ $json['Work email'].toLowerCase().trim() }}` |
| crm_target | `hubspot` (Fixed) |

Square brackets because field names contain spaces. `crm_target` is where you flip CRMs.

**Node 3. Find existing lead** (Data Table → Row → Get). Table `leads`. Condition: `email` equals `{{ $json.email }}`. Settings tab → **Always Output Data: ON** (otherwise an empty result stops the workflow, and new leads would never pass).

**Node 4. Is new lead?** (If). Condition: `{{ $json.isEmpty() }}` → Boolean → is true. True = new, false = duplicate.

**Node 5a. Log duplicate** (Discord, from false branch). Message: `ℹ️ Duplicate lead ignored: {{ $('Normalize lead').first().json.email }}`.

**Node 5b. Restore lead** (Edit Fields, from true branch). The Data Table "get" replaced the lead with an empty item, so pull the original back. Nine fields, each `{{ $('Normalize lead').first().json.<field> }}`.

Why `.first()` and not `.item`? One form submit = one item, so first() is safe and does not depend on item pairing through the Data Table node.

**Node 6. Qualify with AI** (Basic LLM Chain). Prompt: Define below. Toggle **Require Specific Output Format: ON**. Paste:
```
You are an SDR assistant for a B2B SaaS company that sells to companies with 20 to 500 employees who have inbound lead volume problems.

Score this inbound lead.

Lead:
- Name: {{ $json.full_name }}
- Email: {{ $json.email }}
- Company: {{ $json.company }}
- Website: {{ $json.website }}
- Message: {{ $json.message }}

Rules:
- Free email domains (gmail, yahoo, hotmail, outlook) lower the score.
- A message describing a concrete business problem with numbers raises the score.
- Students, agencies looking for partnership, and vendors pitching services are cold.

Return score 1 to 10, tier (hot = 8 to 10, warm = 5 to 7, cold = 1 to 4), icp_fit (true/false), a one-sentence reason, and next_action (one short sentence for the sales rep).
```
Attach two sub-nodes below it:
- **Model**: Google Gemini Chat Model. Model `gemini-2.5-flash`. Options → Temperature `0.2`.
- **Output Parser**: Structured Output Parser. Schema type: Manual. Paste:
  ```json
  {
    "type": "object",
    "properties": {
      "score": { "type": "number" },
      "tier": { "type": "string", "enum": ["hot", "warm", "cold"] },
      "icp_fit": { "type": "boolean" },
      "reason": { "type": "string" },
      "next_action": { "type": "string" }
    },
    "required": ["score", "tier", "icp_fit", "reason", "next_action"]
  }
  ```
Execute step. Output is `{ output: { score, tier, ... } }`.

**Node 7. Merge lead + score** (Edit Fields). The AI node outputs only its answer; the lead fields are gone again. Nine lead fields from `{{ $('Restore lead').item.json.<field> }}` plus five from `{{ $json.output.score }}`, `.tier`, `.icp_fit`, `.reason`, `.next_action`. Set types: score Number, icp_fit Boolean.

**Node 8. Save lead** (Data Table → Row → Upsert). Table `leads`. Match: `email` equals `{{ $json.email }}`. Mapping: Map Automatically.

**Node 9. Route by tier** (Switch). Three rules on `{{ $json.tier }}`: equals `hot` / `warm` / `cold`. Rename outputs.

**Node 10 to 12. Push to CRM × 3** (Execute Sub-workflow). One per Switch output. Workflow: `02 - Push to CRM`. Mode: run once with all items.

Why three identical nodes instead of one? Each branch continues differently after the CRM push. Three nodes keeps the canvas readable. Alternative: one node, then a second Switch. Pick either, document it.

**Node 13. Alert sales (Discord)**, after hot push. Message:
```
🔥 HOT LEAD ({{ $('Merge lead + score').item.json.score }}/10)
{{ $('Merge lead + score').item.json.full_name }} at {{ $('Merge lead + score').item.json.company }}
Email: {{ $('Merge lead + score').item.json.email }}
Says: "{{ $('Merge lead + score').item.json.message }}"
Why: {{ $('Merge lead + score').item.json.reason }}
Next: {{ $('Merge lead + score').item.json.next_action }}
```
Why `$('Merge lead + score')` and not `$json`? Because `$json` here is the sub-workflow's return value (`crm_status`), not the lead.

**Node 14. Follow-up email #1 (Gmail)**, after warm push. To: `{{ $('Merge lead + score').item.json.email }}`. Subject: `Re: {{ $('Merge lead + score').item.json.company }} and your inbound lead process`. Body: see workflow JSON. Keep it plain text, 5 lines, one CTA.

Cold: nothing after the push. Deliberate.

**Settings** → Error workflow → `00 - Error handler`. Publish.

---

## Part D: Item-count checklist

Walk this before every demo. If a number is off, that node is your bug.

| Node | In | Out |
|---|---|---|
| Lead form | 0 | 1 |
| Normalize lead | 1 | 1 |
| Find existing lead | 1 | 1 (empty if new) |
| Is new lead? | 1 | 1 on true OR 1 on false |
| Restore lead | 1 | 1 |
| Qualify with AI | 1 | 1 |
| Merge lead + score | 1 | 1 |
| Save lead | 1 | 1 |
| Route by tier | 1 | 1 on exactly one output |
| Push to CRM | 1 | 1 |
| Alert / Email | 1 | 1 |

---

## Part E: Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Workflow stops silently after Find existing lead | Always Output Data is off | Settings tab of that node → ON |
| Switch sends everything to fallback | AI returned `"Hot"` not `"hot"` | Enum in parser schema is lowercase; add `.toLowerCase()` in Merge node as belt and braces |
| `Cannot read properties of undefined (reading 'score')` | Output parser not attached, so `$json.output` missing | Attach Structured Output Parser, toggle Require Specific Output Format |
| HubSpot 403 | Private app missing contacts.write scope | Add scope, regenerate token |
| Gmail sends from wrong account | OAuth credential signed in as another Google user | Recreate credential |
| Error handler never fires | Main workflow Settings → Error workflow not set | Set it on every workflow |
| Second submit of same email re-scores | Data Table lookup condition typo | Column must be `email`, operator `equals` |
