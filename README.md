# AI-Powered Lead Scoring & Nurture Automation
### Workflow 02 — ActiveCampaign + Zapier + OpenAI

> **Stack:** ActiveCampaign · Zapier · OpenAI (GPT-3.5-turbo) · Slack · Google Sheets  
> **Type:** Lead Scoring, CRM Automation, AI Classification, Multi-Path Routing  
> **Author:** Automation & AI Systems Specialist  
> **Last Updated:** 2026-02

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Workflow Steps](#workflow-steps)
4. [ActiveCampaign Configuration](#activecampaign-configuration)
5. [Zapier Zap Structure](#zapier-zap-structure)
6. [OpenAI Prompt Engineering](#openai-prompt-engineering)
7. [Routing Logic & Paths](#routing-logic--paths)
8. [Email Sequence Specification](#email-sequence-specification)
9. [Output Integrations](#output-integrations)
10. [Error Handling & Edge Cases](#error-handling--edge-cases)
11. [Testing Checklist](#testing-checklist)
12. [Changelog](#changelog)

---

## Overview

This automation eliminates manual lead triage by using AI to score every new contact the moment they enter ActiveCampaign. Based on the AI-generated score, leads are automatically routed into the correct nurture sequence, the CRM is updated, and the sales team is alerted — all within seconds of a new contact being added.

### Problem Solved

| Before | After |
|--------|-------|
| Sales manually reviewed every new lead | AI scores every lead automatically |
| All leads got the same email sequence | 3 personalized sequences based on intent score |
| Hours of delay before outreach | Sub-60 second enrollment from signup |
| No lead quality visibility for ops | Live Google Sheets dashboard with scores |

### Key Metrics This Automation Targets

- **Time-to-first-contact:** Reduced from hours → under 60 seconds
- **Sequence relevance:** 3 tailored tracks vs. 1 generic sequence
- **Sales efficiency:** Only hot leads (score ≥ 70) trigger a Slack alert
- **CRM data quality:** Every contact gets AI-enriched fields on entry

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│              TRIGGER LAYER                              │
│  ActiveCampaign: New Contact Added to "New Leads" List  │
└────────────────────────┬────────────────────────────────┘
                         │ Contact payload (email, name,
                         │ source, title, custom fields)
                         ▼
┌─────────────────────────────────────────────────────────┐
│              ZAPIER ORCHESTRATION                       │
│  Step 1: Receive AC trigger                             │
│  Step 2: Formatter — build prompt string               │
└────────────────────────┬────────────────────────────────┘
                         │ Structured prompt
                         ▼
┌─────────────────────────────────────────────────────────┐
│              AI CLASSIFICATION                          │
│  OpenAI GPT-3.5-turbo                                  │
│  Returns: { score, persona, sequence, reason }         │
└────────────────────────┬────────────────────────────────┘
                         │ JSON response parsed
                         ▼
┌─────────────────────────────────────────────────────────┐
│              CRM UPDATE (AC)                           │
│  Write score, persona tag, sequence to contact record   │
└────────────────────────┬────────────────────────────────┘
                         │ Score evaluated
                         ▼
┌─────────────────────────────────────────────────────────┐
│              ROUTING (Zapier Paths)                    │
│  score ≥ 70 → HOT   │  40-69 → WARM   │  <40 → COLD   │
└──────┬───────────────┴────────┬─────────┴──────┬────────┘
       │                        │                 │
       ▼                        ▼                 ▼
  🔥 Hot Track           ⚡ Warm Track        ❄️ Cold Track
  AC Sequence            AC Sequence          AC Sequence
  Slack Alert            Slack Digest         No Alert
  Sheets Log             Sheets Log           Sheets Log
```

---

## Workflow Steps

### Step 1 — ActiveCampaign Trigger

**App:** ActiveCampaign  
**Event:** New Contact Added to List  
**List Name:** `New Leads`

This trigger fires whenever a contact is added to the `New Leads` list, whether through:
- A web form submission
- A manual import
- An API call from a third-party tool
- Another AC automation

**Fields passed to Zapier:**

| Field | AC Field Name | Notes |
|-------|---------------|-------|
| Email | `email` | Required |
| First Name | `first_name` | Used in email personalization |
| Last Name | `last_name` | Used in email personalization |
| Job Title | `field_title` | Custom field — used in AI scoring |
| Company | `field_company` | Custom field — used in AI scoring |
| Lead Source | `field_source` | e.g. webinar, paid-ad, organic |
| Signup Date | `cdate` | Timestamp of contact creation |

---

### Step 2 — Zapier Formatter

**App:** Zapier Formatter  
**Action:** Text — Construct  

Builds a clean string from AC contact fields to use as the AI prompt body. This ensures consistent formatting regardless of which fields are populated.

**Template used:**
```
Contact Name: {{first_name}} {{last_name}}
Email: {{email}}
Job Title: {{field_title}}
Company: {{field_company}}
Lead Source: {{field_source}}
```

---

### Step 3 — OpenAI Lead Scoring

**App:** OpenAI (via Zapier)  
**Action:** Send Prompt  
**Model:** gpt-3.5-turbo  

See [OpenAI Prompt Engineering](#openai-prompt-engineering) section for full prompt.

---

### Step 4 — ActiveCampaign Contact Update

**App:** ActiveCampaign  
**Action:** Update Contact  

Writes AI output back to the contact's custom fields:

| Custom Field | Value Written | Example |
|-------------|---------------|---------|
| `ai_lead_score` | score (integer) | `82` |
| `ai_persona` | persona string | `decision_maker` |
| `ai_sequence` | sequence name | `high-intent-fast-track` |
| `ai_score_reason` | reason string | `C-suite title, enterprise domain` |

---

### Step 5 — Zapier Paths (Router)

**App:** Zapier Paths  
**Branches:** 3 (see [Routing Logic](#routing-logic--paths))

---

### Step 6 — Multi-Output Actions

Run in parallel within each path:
1. **ActiveCampaign:** Add contact to the correct email sequence
2. **Slack:** Post formatted alert to the correct channel  
3. **Google Sheets:** Append row to the lead scoring dashboard

---

## ActiveCampaign Configuration

### Lists Required

| List Name | Purpose |
|-----------|---------|
| `New Leads` | Intake list — triggers this Zap |
| `Hot Leads` | Receives contacts with score ≥ 70 |
| `Warm Nurture` | Receives contacts with score 40–69 |
| `Cold Drip` | Receives contacts with score < 40 |

### Custom Fields Required

Navigate to: **Lists → Manage Fields → Add Field**

| Field Label | Field Type | Zapier Key |
|-------------|-----------|------------|
| AI Lead Score | Number | `ai_lead_score` |
| AI Persona | Text | `ai_persona` |
| AI Sequence | Text | `ai_sequence` |
| AI Score Reason | Textarea | `ai_score_reason` |
| Lead Source | Text | `field_source` |
| Job Title | Text | `field_title` |
| Company | Text | `field_company` |

### Tags Used

```
lead-score-high       # score ≥ 70
lead-score-medium     # score 40-69
lead-score-low        # score < 40
persona-decision-maker
persona-influencer
persona-researcher
persona-practitioner
sequence-hot-track
sequence-standard-nurture
sequence-cold-drip
```

### Automations Required in ActiveCampaign

You need 3 AC automations, each triggered by a tag:

**Automation 1: High-Intent Fast Track**
- Trigger: Tag added = `sequence-hot-track`
- Emails: 5-step sequence (see Email Sequence Specification)
- Goal: Book a demo call

**Automation 2: Standard Nurture**
- Trigger: Tag added = `sequence-standard-nurture`
- Emails: 7-step sequence over 3 weeks
- Goal: Educate and warm up

**Automation 3: Cold Drip**
- Trigger: Tag added = `sequence-cold-drip`
- Emails: Monthly touchpoints for 90 days
- Goal: Stay top of mind, re-score at Day 30

---

## Zapier Zap Structure

### Zap Name
`AC Lead Scorer — AI Routing v1`

### Step-by-Step Zapier Configuration

```
[TRIGGER]
App:    ActiveCampaign
Event:  New Contact Added to List
List:   New Leads
Test:   Add a test contact to "New Leads" list

[ACTION 1]
App:    Formatter by Zapier
Event:  Text → Construct
Input:  Map AC fields into prompt template

[ACTION 2]
App:    OpenAI (ChatGPT)
Event:  Send Prompt
Model:  gpt-3.5-turbo
Prompt: [See prompt below]

[ACTION 3]
App:    ActiveCampaign
Event:  Update Contact
Match:  By email (from step 1)
Fields: ai_lead_score, ai_persona, ai_sequence, ai_score_reason

[ACTION 4]
App:    Paths by Zapier
Path A: score contains ≥ 70 → HOT
Path B: score contains 40-69 → WARM
Path C: All other → COLD

[Within Path A]
  → AC: Add Tag "lead-score-high"
  → AC: Add Tag "sequence-hot-track"
  → AC: Subscribe to "Hot Leads" list
  → Slack: Post to #sales-hot-leads
  → Google Sheets: Create Row

[Within Path B]
  → AC: Add Tag "lead-score-medium"
  → AC: Add Tag "sequence-standard-nurture"
  → AC: Subscribe to "Warm Nurture" list
  → Slack: Post to #sales-pipeline
  → Google Sheets: Create Row

[Within Path C]
  → AC: Add Tag "lead-score-low"
  → AC: Add Tag "sequence-cold-drip"
  → AC: Subscribe to "Cold Drip" list
  → Google Sheets: Create Row (no Slack alert)
```

---

## OpenAI Prompt Engineering

### System Prompt

```
You are a B2B lead scoring specialist. Your job is to evaluate inbound leads 
and return a structured JSON score based on their profile data.

Scoring criteria:
- Job title seniority: C-suite/VP/Director = higher score
- Company domain: enterprise/known brand = higher score
- Lead source: demo-request/referral = higher than newsletter/organic
- Title relevance: decision-maker roles score higher than practitioners

Return ONLY a JSON object. No explanation, no markdown, no preamble.
```

### User Prompt (with mapped Zapier fields)

```
Score the following lead. Return JSON only.

Contact:
Name: [FIRST_NAME] [LAST_NAME]
Email: [EMAIL]
Job Title: [FIELD_TITLE]
Company: [FIELD_COMPANY]
Lead Source: [FIELD_SOURCE]

Return this exact JSON structure:
{
  "score": <integer 1-100>,
  "persona": "<decision_maker|influencer|practitioner|researcher>",
  "sequence": "<high-intent-fast-track|standard-nurture|cold-drip>",
  "reason": "<one sentence explanation>"
}
```

### Parsing the Response in Zapier

After the OpenAI step, add a **Code by Zapier** step (JavaScript) to safely parse the JSON:

```javascript
const raw = inputData.ai_response;
// Strip markdown code fences if present
const cleaned = raw.replace(/```json|```/g, '').trim();
const parsed = JSON.parse(cleaned);

return {
  score: parsed.score,
  persona: parsed.persona,
  sequence: parsed.sequence,
  reason: parsed.reason
};
```

---

## Routing Logic & Paths

### Path Conditions

| Path | Name | Condition | AC Sequence | Slack Channel |
|------|------|-----------|-------------|---------------|
| A | 🔥 Hot Lead | `score >= 70` | High-Intent Fast Track | #sales-hot-leads |
| B | ⚡ Warm Lead | `score >= 40` AND `score < 70` | Standard Nurture | #sales-pipeline |
| C | ❄️ Cold Lead | `score < 40` | Cold Drip | *(no alert)* |

> **Note:** Zapier Paths evaluate top-to-bottom. Path A must be checked before Path B to avoid warm leads matching both.

### Zapier Path Condition Setup

**Path A:**
```
(Only continue if) ai_score (Number) Greater than or equal to 70
```

**Path B:**
```
(Only continue if) ai_score (Number) Greater than or equal to 40
AND ai_score (Number) Less than 70
```

**Path C:**
```
(Only continue if) [No condition — this is the fallback/else path]
```

---

## Email Sequence Specification

### Hot Lead Track — 5 Emails

| # | Email | Trigger | Subject Line Strategy | CTA |
|---|-------|---------|----------------------|-----|
| 1 | Welcome | Immediate | `[First Name], quick note from [Company]` | Book a 15-min call |
| 2 | Case Study | +24 hrs (if E1 opened) | `How [Similar Company] solved [pain point]` | Read the story |
| 3 | Demo Invite | +48 hrs (if link clicked) | `Want to see this live, [First Name]?` | Book a demo |
| 4 | Re-Score | +72 hrs (if no booking) | Zapier webhook re-runs AI scoring | — |
| 5 | Breakup | +5 days (if no response) | `Should I stop sending these?` | One last CTA |

### Conditional Logic Within AC Sequence

```
Email 1 sent
  └─ IF opened → send Email 2 (24hr wait)
  └─ IF NOT opened → resend Email 1 with alternate subject (12hr wait)
       └─ IF still not opened → skip to Email 3

Email 2 sent
  └─ IF link clicked → send Email 3 (48hr wait)
  └─ IF NOT clicked → send value-add content instead

Email 3 sent
  └─ IF demo booked (Goal met) → exit sequence, add tag "demo-booked"
  └─ IF NOT booked → wait 72hrs → fire Zapier webhook for re-score

Email 5 sent
  └─ IF opened → keep in hot track
  └─ IF NOT opened → add tag "re-engagement-needed", move to 90-day drip
```

---

## Output Integrations

### Slack Message Format

**Channel:** `#sales-hot-leads` (Hot), `#sales-pipeline` (Warm)

```
🔥 HOT LEAD ALERT — Score: {{ai_score}}/100

👤 {{first_name}} {{last_name}}
💼 {{field_title}} @ {{field_company}}
📧 {{email}}
🔗 Source: {{field_source}}

🤖 AI Persona: {{ai_persona}}
📋 Reason: {{ai_score_reason}}
📨 Enrolled in: {{ai_sequence}}

→ View in ActiveCampaign: https://app.ac-link.com/contact/{{ac_contact_id}}
```

### Google Sheets Log — Column Schema

| Column | Source | Notes |
|--------|--------|-------|
| Timestamp | Zapier auto | ISO 8601 |
| First Name | AC field | |
| Last Name | AC field | |
| Email | AC field | |
| Job Title | AC field | |
| Company | AC field | |
| Lead Source | AC field | |
| AI Score | OpenAI output | Integer 1–100 |
| AI Persona | OpenAI output | |
| AI Sequence | OpenAI output | |
| AI Reason | OpenAI output | |
| Score Tier | Calculated | HOT / WARM / COLD |
| Zapier Run ID | Zapier auto | For debugging |

---

## Error Handling & Edge Cases

### Scenario 1: OpenAI Returns Non-JSON

**Cause:** Model adds explanation text or markdown fences  
**Fix:** The JavaScript parser step strips fences. If `JSON.parse()` throws, Zapier will halt the Zap and send an error email. Add a fallback by catching the error and defaulting:

```javascript
try {
  const parsed = JSON.parse(cleaned);
  return { score: parsed.score, persona: parsed.persona, ... };
} catch (e) {
  return { score: 50, persona: 'unknown', sequence: 'standard-nurture', reason: 'Parse error — defaulted' };
}
```

### Scenario 2: Missing Contact Fields

**Cause:** Contact added with only email (no title, company)  
**Fix:** AI prompt is designed to still return a valid score with partial data. Score will be lower by default (no title = lower seniority signal). The sequence will default to `standard-nurture`.

### Scenario 3: Duplicate Contact Added

**Cause:** Same email re-added to "New Leads" list  
**Fix:** In AC, enable "Skip contacts already in list." Additionally, add a Zapier Filter before Step 3: check if `ai_lead_score` field is already populated — if yes, skip the run.

### Scenario 4: Zapier Rate Limits

**Cause:** Large import floods the trigger  
**Fix:** Use Zapier's built-in delay. For imports over 100 contacts, use AC's native bulk automation instead and only use this Zap for real-time single additions.

---

## Testing Checklist

Use this checklist before going live:

```
SETUP
[ ] "New Leads" list created in ActiveCampaign
[ ] All 7 custom fields created and mapped
[ ] 3 AC automations created (one per sequence)
[ ] All required tags created in AC
[ ] Slack channels created: #sales-hot-leads, #sales-pipeline
[ ] Google Sheet created with correct column headers
[ ] OpenAI API key connected in Zapier

ZAP STEPS
[ ] Trigger tested — test contact added and Zapier received it
[ ] Formatter step produces clean prompt string
[ ] OpenAI returns valid JSON (check with 3 test contacts)
[ ] AC update writes score correctly to custom fields
[ ] Paths route correctly (test one contact per tier)
[ ] Slack message posts with correct data
[ ] Google Sheets row appended correctly

EDGE CASES
[ ] Test with missing job title — does AI still score?
[ ] Test with score exactly 70 — goes to HOT (not WARM)?
[ ] Test with score exactly 40 — goes to WARM (not COLD)?
[ ] Test duplicate contact — is it skipped?
[ ] Test OpenAI failure — does fallback score apply?

SEQUENCE VALIDATION
[ ] Hot track email 1 sends immediately on enrollment
[ ] Conditional logic (if opened / if not) works correctly
[ ] Re-score webhook fires at Day 3
[ ] Breakup email sends at Day 5 only if no prior response
```

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| v1.0 | 2026-02 | Initial build — 3-path routing, 5-email hot sequence |
| — | — | Planned: Add industry detection to AI prompt |
| — | — | Planned: Re-score loop at 30-day mark for cold leads |
| — | — | Planned: HubSpot output option as alternative to Sheets |

---

## Related Files

```
/
├── README.md                          ← This file
├── process-map-ac-workflow.html       ← Interactive process map (open in browser)
├── process-map-ticket-router.html     ← Workflow 01: AI Support Ticket Router
└── zapier-prompt-template.txt         ← Copy-paste OpenAI prompt for Zapier
```

---

*Built for interview demonstration purposes. All logic mirrors production-grade automation patterns used in Intercom, HubSpot, and Salesforce implementations.*
