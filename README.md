# AI Grant & Scholarship Discovery Agent

An autonomous three-workflow system built in n8n that continuously monitors scholarship, internship, and grant opportunities from multiple RSS sources, scores them against a student profile using AI, and delivers personalized alerts and weekly digests.

## Live System
- Workflow 1 runs every 6 hours automatically
- Workflow 2 sends a weekly digest every Sunday at 6pm
- Workflow 3 sends daily deadline alerts every morning at 8am

## Workflows

### Workflow 1: Ingestion Pipeline
Runs every 6 hours. Fetches from 2 RSS sources, deduplicates using MD5 hashing, and runs three sequential AI reasoning steps before saving.

**Node-by-node pipeline:**
Schedule Trigger (every 6 hours)
│
├──────────────────────────────┐
↓                              ↓
RSS Read                           RSS Read1
(opportunitydesk.org/feed)         (opportunitiesfeed.com/feed)
│                              │
└──────────┬───────────────────┘
↓
Merge1 (Append mode)
↓
Code in JavaScript (Normalize fields)
↓
Generate Hash (MD5 on title+org)
↓
Check Duplicate (Google Sheets lookup by hash)
↓
Dedup Check (gate: new item or duplicate?)
│
┌───────┴────────┐
Duplicate           New item
(stop)               ↓
Get Profile
↓
Check Eligibility (Groq API)
↓
Parse Eligibility + extract deadline via regex
↓
Is Eligible (IF node)
┌────────┴────────┐
Not Eligible        Eligible
↓                  ↓
Append row in sheet    Build Score Fit Body
(status: NotEligible)       ↓
Score Fit (Groq API)
↓
Parse Fit Score
↓
Fit Threshold (IF: score ≥ 70)
┌──────────┴──────────┐
Low Fit              High Fit
↓                    ↓
Low Fit node       Build Checklist Body
(status: LowFit)             ↓
Generate Checklist (Groq API)
↓
Parse Checklist
↓
Save Opportunity
(status: New)
**AI prompts (3 separate structured JSON outputs):**
- **Check Eligibility** → `{"eligible": bool, "reason": string, "missing_requirements": []}`
- **Score Fit** → `{"fit_score": 0-100, "rationale": string}`
- **Generate Checklist** → `{"checklist": string, "resume_gaps": string}`

**Deadline extraction:** regex patterns on raw RSS description text, handles multiple date formats and "Rolling Basis"/"Ongoing" cases.

---

### Workflow 2: Weekly Digest
Runs every Sunday at 6pm.
Schedule Trigger (weekly)
↓
Get row(s) in sheet (filter: status = New)
↓
HasNew Opportunities (IF: title not empty)
↓
Filter Quality Matches (eligible = true AND fit-score ≥ 70)
↓
Build Email HTML (Code node: formats all items into HTML)  ──→  Mark As Notified
↓                                                         (update status → Notified)
Send Digest Email (Gmail)
---

### Workflow 3: Deadline Alerts
Runs every day at 8am.
Schedule Trigger (daily at 8am)
↓
Get Upcoming Deadline (all rows from Opportunities sheet)
↓
Filter Upcoming Deadline (Code node: deadline within next 7 days, skips Rolling/Ongoing)
↓
Send Deadline Alert (Gmail, one email per opportunity)
---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow Automation | n8n |
| AI / LLM | Groq API — Llama 3.3-70B Versatile |
| Storage | Google Sheets (OAuth2) |
| Email | Gmail (OAuth2) |
| Hashing | MD5 via Node.js crypto module |
| Sources | RSS Feed Read nodes |

## Data Sources
- **OpportunityDesk** (`opportunitydesk.org/feed`) — Scholarships, fellowships, grants, accelerators
- **OpportunitiesFeed** (`opportunitiesfeed.com/feed`) — Jobs, internships, scholarships, remote careers

## Google Sheets Schema

### Profile Sheet
One row storing the student profile used for all AI reasoning:
↓
Save Opportunity
(status: New)
