# AI-Powered Lead Scoring & Nurture Automation
### Workflow  — ActiveCampaign + Zapier + AI by Zapier

> **Stack:** ActiveCampaign · Code by Zapier · AI by Zapier · Filter by Zapier · Paths by Zapier · Slack · Google Sheets
> **Total Steps:** 18
> **Version:** v3 — Loop Fixed
> **Last Updated:** 2026-02

---

## ⚠️ Loop Fix (v3)

### What Was Happening
Using **"New or Updated Contact"** as the trigger caused an infinite loop:

```
Step 1: Trigger fires on new contact ✅
Step 6: Writes AI Score back to the contact record
         ↓
Step 1: Fires AGAIN (contact was "updated") 🔁
Step 6: Writes AI Score again
         ↓
Step 1: Fires AGAIN... ♾️ infinite loop
         = duplicate Slack messages + duplicate Sheets rows
```

### The Fix — Two Changes Made

**Change 1: Trigger → "New Contact"**
Only fires once when a contact is first created.
Will not re-trigger when Step 6 updates the contact record.

**Change 2: Step 5 — Double Filter (Safety Net)**

| Condition | Logic |
|-----------|-------|
| Contact ID (Step 4) **Exists** | Confirms contact was found in AC |
| AI Lead Score (Step 4) **Does Not Exist** | Confirms not yet scored |

Both must be TRUE to continue. If AI Score already has a value
the Zap stops silently — no error, no duplicate entry.

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

This automation scores every new contact in ActiveCampaign using AI,
then routes them into the correct nurture sequence within seconds.

### Problem Solved

| Before | After |
|--------|-------|
| Manual lead review | AI scores every contact automatically |
| One generic email sequence | 3 personalized tracks based on AI score |
| Hours before first outreach | Sub-60 second enrollment from trigger |
| Duplicate messages from loop | Fixed — fires once per new contact only |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────┐
│  STEP 1 — ActiveCampaign                             │
│  Trigger: New Contact ← CHANGED from "New or Updated"│
│  Fires ONCE on creation only — loop safe ✅         │ 
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 2 — Code by Zapier                             │
│  Run Javascript — builds prompt string               │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 3 — AI by Zapier                               │
│  Analyze and Return Data                             │
│  ✅ Auto-parses JSON — no extra step needed          │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 4 — ActiveCampaign                             │
│  Find Contact (by email)                             │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 5 — Filter by Zapier  ← UPDATED: DOUBLE FILTER │
│  Condition 1: Contact ID       → Exists              │
│  Condition 2: AI Lead Score    → Does Not Exist      │
│  ↳ Either false = END (silent stop, no duplicate)    │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 6 — ActiveCampaign                             │
│  Create or Update Contact                            │
│  Writes: AI Score, Persona, Sequence, Reason         │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│  STEP 7 — Paths by Zapier                            │
│  Split into 3 paths by AI Score                      │
└──────┬──────────────┴──────────┬─────────────────────┘
       │                         │                    │
  score ≥ 70               score 40-69           score < 40
       │                         │                    │
🔥 HOT LEAD             ⚡ WARM LEAD           ❄️ COLD LEAD
Steps 8-11              Steps 12-15            Steps 16-18
  8.  Path cond.          12. Path cond.         16. Path cond.
  9.  Slack alert         13. Slack alert        17. Google Sheets
  10. Google Sheets       14. Google Sheets      18. AC Tag
  11. AC Tag              15. AC Tag                 (no Slack)
```

---

## Complete Zap Structure

| # | App | Action | Notes |
|---|-----|--------|-------|
| 1 | ActiveCampaign | **New Contact** | Trigger — fires once on creation only ✅ |
| 2 | Code by Zapier | Run Javascript | Builds prompt string from contact fields |
| 3 | AI by Zapier | Analyze and Return Data | Scores lead — auto-parses JSON |
| 4 | ActiveCampaign | Find Contact | Searches by email |
| 5 | Filter by Zapier | Filter conditions | **Double filter** — contact exists AND not yet scored ✅ |
| 6 | ActiveCampaign | Create or Update Contact | Writes AI score, persona, sequence, reason |
| 7 | Paths by Zapier | Split into paths | Routes by score ≥70 / 40-69 / <40 |
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

### Step 1 — ActiveCampaign Trigger

- **App:** ActiveCampaign
- **Event:** New Contact *(NOT "New or Updated Contact")*
- **Why:** Only fires once on creation — prevents loop when Step 6 writes back to AC

---

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

AI by Zapier automatically parses JSON into individual fields.
No separate parser step needed.

**Output fields:**

| Field | Example Value |
|-------|--------------|
| Score | 75 |
| Persona | decision_maker |
| Sequence | high-intent-fast-track |
| Reason | VP title with enterprise domain |

---

### Step 4 — ActiveCampaign Find Contact

- **Search by:** Email → map from Step 1

---

### Step 5 — Filter by Zapier (Double Filter)

| # | Field | Condition | Value |
|---|-------|-----------|-------|
| 1 | Contact ID (Step 4) | Exists | *(blank)* |
| 2 | AI Lead Score (Step 4) | Does not exist | *(blank)* |

**Logic:** Both conditions must be TRUE to continue.

| Scenario | Result |
|----------|--------|
| Contact found + no score yet | ✅ Continues |
| Contact not found | 🛑 Stops silently |
| Contact already has AI score | 🛑 Stops silently — prevents re-scoring |

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

```
You are a B2B lead scoring specialist. Evaluate the lead and return JSON only.
No explanation, no markdown, no extra text.

Scoring criteria:
- Job title seniority: C-suite/VP/Director = higher score
- Company domain: enterprise/known brand = higher score
- Lead source: demo-request/referral = higher than newsletter
- Decision-maker roles score higher than practitioners

[map → prompt output from Code by Zapier Step 2]

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

Timestamp · First Name · Last Name · Email · Job Title ·
Company · Lead Source · AI Score · AI Persona ·
AI Sequence · AI Reason · Tier

---

## Error Handling

| Scenario | Result |
|----------|--------|
| Contact not in AC | Step 5 Filter stops Zap silently |
| Contact already scored | Step 5 Filter stops Zap — no duplicate |
| Missing contact fields | Step 2 uses 'Not provided' fallback |
| Step 6 updates contact | Does NOT re-trigger Zap (New Contact trigger) |

---

## Testing Checklist

```
LOOP FIX VERIFICATION
[ ] Trigger is set to "New Contact" (not "New or Updated Contact")
[ ] Step 5 has TWO filter conditions (Contact ID exists + AI Score does not exist)
[ ] Add test contact → check only ONE Slack message received
[ ] Check Google Sheets → only ONE row per contact

FULL ZAP STEPS
[ ] Step 1: New contact added → Zapier receives it
[ ] Step 2: Code outputs clean prompt string
[ ] Step 3: AI returns Score, Persona, Sequence, Reason as separate fields
[ ] Step 4: Find Contact returns correct Contact ID
[ ] Step 5: Filter passes when contact exists AND has no score
[ ] Step 5: Filter STOPS when contact already has AI score (re-run test)
[ ] Step 6: Custom fields updated in AC contact record
[ ] Steps 8-11:  Score 82 → Hot path → Slack + Sheets + Tag
[ ] Steps 12-15: Score 55 → Warm path → Slack + Sheets + Tag
[ ] Steps 16-18: Score 28 → Cold path → Sheets + Tag only (no Slack)
[ ] Zap published and turned ON
[ ] All 18 steps show green checkmarks
```

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| v3.0 | 2026-02 | Fixed infinite loop — changed trigger to "New Contact" |
| v3.0 | 2026-02 | Step 5 updated to double filter (contact exists + not yet scored) |
| v2.0 | 2026-02 | Rebuilt to match actual 18-step Zap |
| v2.0 | 2026-02 | AI by Zapier replaces OpenAI — auto-parses JSON |
| v1.0 | 2026-02 | Initial draft |

---

## Related Files

```
/
├── README.md                            ← This file
├── process-map-ac-workflow-v3.html      ← Interactive process map (v3)
├── wf02-v3-loop-fixed.png               ← Excalidraw-style diagram (v3)
└── zapier-prompt-template.txt           ← Prompt & config reference
```

*v3 — Loop fixed and tested. 18 steps confirmed working.*
