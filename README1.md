# AI-Powered Lead Scoring & Nurture Automation
### Workflow 02 — ActiveCampaign + Zapier + AI by Zapier

> **Stack:** ActiveCampaign · Code by Zapier · AI by Zapier · Filter by Zapier · Paths by Zapier · Slack · Google Sheets
> **Total Steps:** 18
> **Type:** Lead Scoring, CRM Automation, AI Classification, Multi-Path Routing
> **Last Updated:** 2026-02

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Complete Zap Structure — All 18 Steps](#complete-zap-structure)
4. [Step-by-Step Configuration](#step-by-step-configuration)
5. [AI Prompt Engineering](#ai-prompt-engineering)
6. [Routing Logic & Paths](#routing-logic--paths)
7. [Output Integrations](#output-integrations)
8. [Error Handling](#error-handling)
9. [Testing Checklist](#testing-checklist)
10. [Changelog](#changelog)

---

## Overview

This automation scores every new or updated contact in ActiveCampaign using AI, then routes them into the correct nurture sequence — all within seconds. No manual triage. No missed leads.

### Problem Solved

| Before | After |
|--------|-------|
| Manual lead review | AI scores every contact automatically |
| One generic email sequence | 3 personalized tracks based on AI score |
| Hours before first outreach | Sub-60 second enrollment from trigger |
| No lead quality data in CRM | AI score, persona & reason written to AC |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────┐
│  STEP 1 — ActiveCampaign                     │
│  Trigger: New or Updated Contact             │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 2 — Code by Zapier                     │
│  Run Javascript — builds prompt string       │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 3 — AI by Zapier                       │
│  Analyze and Return Data                     │
│  ✅ Auto-parses JSON — no extra step needed  │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 4 — ActiveCampaign                     │
│  Find Contact (by email)                     │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 5 — Filter by Zapier                   │
│  Only continue if Contact ID exists          │
│  ↳ If not found → END (silent stop)          │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 6 — ActiveCampaign                     │
│  Create or Update Contact                    │
│  Writes: AI Score, Persona, Sequence, Reason │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│  STEP 7 — Paths by Zapier                    │
│  Split into 3 paths by AI Score              │
└──────┬──────────┴─────────┬──────────────────┘
       │                    │                 │
  score ≥ 70           score 40-69        score < 40
       │                    │                 │
🔥 HOT LEAD          ⚡ WARM LEAD        ❄️ COLD LEAD
Steps 8-11           Steps 12-15         Steps 16-18
  8.  Path cond.      12. Path cond.     16. Path cond.
  9.  Slack alert     13. Slack alert    17. Google Sheets
  10. Google Sheets   14. Google Sheets  18. AC Tag
  11. AC Tag          15. AC Tag             (no Slack)
```

---

## Complete Zap Structure

| # | App | Action | Notes |
|---|-----|--------|-------|
| 1 | ActiveCampaign | New or Updated Contact | Trigger |
| 2 | Code by Zapier | Run Javascript | Builds prompt string |
| 3 | AI by Zapier | Analyze and Return Data | Scores lead, auto-parses JSON |
| 4 | ActiveCampaign | Find Contact | Search by email |
| 5 | Filter by Zapier | Filter conditions | Only if contact exists |
| 6 | ActiveCampaign | Create or Update Contact | Writes AI fields to CRM |
| 7 | Paths by Zapier | Split into paths | Routes by AI score |
| 8 | Paths | Path conditions | Hot Lead — score ≥ 70 |
| 9 | Slack | Send Channel Message | #sales-hot-leads |
| 10 | Google Sheets | Create Spreadsheet Row | Tier: HOT |
| 11 | ActiveCampaign | Add/Remove Tag From Contact | sequence-hot-track |
| 12 | Paths | Path conditions | Warm Lead — score 40-69 |
| 13 | Slack | Send Channel Message | #sales-pipeline |
| 14 | Google Sheets | Create Spreadsheet Row | Tier: WARM |
| 15 | ActiveCampaign | Add/Remove Tag From Contact | sequence-standard-nurture |
| 16 | Paths | Path conditions | Cold Lead — fallback |
| 17 | Google Sheets | Create Spreadsheet Row | Tier: COLD |
| 18 | ActiveCampaign | Add/Remove Tag From Contact | sequence-cold-drip |

---

## Step-by-Step Configuration

### Step 2 — Code by Zapier (Javascript)

**Input Data:**

| Key | Value (map from Step 1) |
|-----|------------------------|
| firstName | Contact First Name |
| lastName | Contact Last Name |
| email | Contact Email Address |
| jobTitle | Job Title |
| company | Company |
| leadSource | Lead Source |

**Code:**

```javascript
output = {
  prompt: `Contact Name: ${inputData.firstName || 'Not provided'} ${inputData.lastName || ''}
Email: ${inputData.email || 'Not provided'}
Job Title: ${inputData.jobTitle || 'Not provided'}
Company: ${inputData.company || 'Not provided'}
Lead Source: ${inputData.leadSource || 'Not provided'}`
};
```

---

### Step 3 — AI by Zapier

> AI by Zapier automatically parses the JSON response into individual fields.
> No separate Code parser step is needed.

**Output fields:**

| Field | Example |
|-------|---------|
| Score | 75 |
| Persona | decision_maker |
| Sequence | high-intent-fast-track |
| Reason | VP title with enterprise domain |

---

### Step 5 — Filter by Zapier

| Field | Condition | Value |
|-------|-----------|-------|
| Contact ID (Step 4) | Exists | *(leave blank)* |

---

### Step 6 — Create or Update Contact

| Custom Field | Map From |
|-------------|----------|
| AI Lead Score | Score → Step 3 |
| AI Persona | Persona → Step 3 |
| AI Sequence | Sequence → Step 3 |
| AI Reason | Reason → Step 3 |

---

## AI Prompt Engineering

### Prompt (paste into AI by Zapier)

```
You are a B2B lead scoring specialist. Evaluate the lead and return JSON only.
No explanation, no markdown, no extra text.

Scoring criteria:
- Job title seniority: C-suite/VP/Director = higher score
- Company domain: enterprise/known brand = higher score
- Lead source: demo-request/referral = higher than newsletter
- Decision-maker roles score higher than practitioners

[map → prompt output from Step 2]

Return exactly this JSON:
{
  "score": <integer 1-100>,
  "persona": "<decision_maker|influencer|practitioner|researcher>",
  "sequence": "<high-intent-fast-track|standard-nurture|cold-drip>",
  "reason": "<one sentence>"
}
```

---

## Routing Logic & Paths

| Path | Condition | Slack | Sheets Tier | AC Tag |
|------|-----------|-------|-------------|--------|
| 🔥 Hot | Score ≥ 70 | #sales-hot-leads | HOT | sequence-hot-track |
| ⚡ Warm | Score ≥ 40 | #sales-pipeline | WARM | sequence-standard-nurture |
| ❄️ Cold | Fallback | None | COLD | sequence-cold-drip |

> Zapier evaluates paths top-to-bottom. Hot is checked first so scores ≥ 70 never reach Warm.

---

## Output Integrations

### Slack Message

```
🔥 HOT LEAD — Score: [Score]/100
👤 [First Name] [Last Name]
💼 [Job Title] @ [Company]
📧 [Email]
🤖 Persona: [Persona]
📋 Reason: [Reason]
📨 Sequence: [Sequence]
```

### Google Sheets Columns

Timestamp · First Name · Last Name · Email · Job Title · Company · Lead Source · AI Score · AI Persona · AI Sequence · AI Reason · Tier

---

## Error Handling

| Scenario | Result |
|----------|--------|
| Contact not found in AC | Step 5 Filter stops Zap silently |
| Missing contact fields | Step 2 uses 'Not provided' fallback |
| Duplicate contact trigger | Step 6 updates existing record |

---

## Testing Checklist

```
[ ] Step 1: Test contact added → Zapier receives it
[ ] Step 2: Code outputs clean prompt string
[ ] Step 3: AI returns Score, Persona, Sequence, Reason as separate fields
[ ] Step 4: Find Contact returns correct Contact ID
[ ] Step 5: Filter passes when contact exists; stops silently when not found
[ ] Step 6: Custom fields updated in AC contact record
[ ] Steps 8-11: Score 82 → Hot path → Slack + Sheets + Tag
[ ] Steps 12-15: Score 55 → Warm path → Slack + Sheets + Tag
[ ] Steps 16-18: Score 28 → Cold path → Sheets + Tag only (no Slack)
[ ] Zap published and turned ON
[ ] All 18 steps show green checkmarks on live test
```

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| v2.0 | 2026-02 | Rebuilt to match actual 18-step Zap |
| v2.0 | 2026-02 | Replaced OpenAI with AI by Zapier (auto-parses JSON) |
| v2.0 | 2026-02 | Added Find Contact + Filter safety gate (Steps 4-5) |
| v2.0 | 2026-02 | Cold path confirmed — no Slack alert |
| v1.0 | 2026-02 | Initial draft |

---

## Related Files

```
/
├── README.md                        ← This file
├── process-map-ac-workflow.html     ← Interactive process map
├── wf02-final-diagram.png           ← Excalidraw-style diagram
└── zapier-prompt-template.txt       ← AI prompt copy-paste reference
```

*Built and tested in Zapier. 18 steps confirmed working.*
