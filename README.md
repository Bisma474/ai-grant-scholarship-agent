# AI Grant & Scholarship Discovery Agent

An autonomous three-workflow system built in n8n that continuously monitors scholarship, internship, and grant opportunities from multiple RSS sources, scores them against a student profile using AI, and delivers personalized alerts and weekly digests.

## Live System

- Workflow 1 runs every 6 hours automatically
- Workflow 2 sends a weekly digest every Sunday at 6pm
- Workflow 3 sends daily deadline alerts every morning at 8am

---

## Workflows

### Workflow 1: Ingestion Pipeline

Runs every 6 hours. Fetches from 2 RSS sources, deduplicates using MD5 hashing, and runs three sequential AI reasoning steps before saving.

**Pipeline steps:**

1. Schedule Trigger (every 6 hours) fires RSS Read (OpportunityDesk) and RSS Read1 (OpportunitiesFeed) in parallel
2. Merge1 appends both feeds into one list
3. Code in JavaScript normalizes fields (title, org, url, description, rawDate)
4. Generate Hash creates an MD5 hash from title + org
5. Check Duplicate looks up the hash in Google Sheets
6. Dedup Check gates the item — duplicates stop here, new items continue
7. Get Profile fetches student profile from Google Sheets
8. Check Eligibility calls Groq API (Llama 3.3-70B), returns structured JSON
9. Parse Eligibility parses the response and extracts deadline via regex
10. Is Eligible branches the item:
    - **Not Eligible** → saved to sheet with status `Not Eligible`, pipeline stops
    - **Eligible** → continues to scoring
11. Build Score Fit Body constructs the scoring prompt with resume and opportunity data
12. Score Fit calls Groq API, returns fit score 0-100
13. Parse Fit Score parses the response
14. Fit Threshold branches on score ≥ 70:
    - **Low Fit** → saved to sheet with status `LowFit`, pipeline stops
    - **High Fit** → continues to checklist generation
15. Build Checklist Body constructs the checklist prompt
16. Generate Checklist calls Groq API, returns checklist and resume gap analysis
17. Parse Checklist parses the response
18. Save Opportunity appends final row to sheet with status `New`

**AI prompts — 3 separate structured JSON outputs:**

| Step | Output Schema |
|---|---|
| Check Eligibility | `{"eligible": bool, "reason": string, "missing_requirements": []}` |
| Score Fit | `{"fit_score": 0-100, "rationale": string}` |
| Generate Checklist | `{"checklist": string, "resume_gaps": string}` |

**Deadline extraction:** regex patterns on raw RSS description text, handles multiple date formats and Rolling Basis / Ongoing cases.

---

### Workflow 2: Weekly Digest

Runs every Sunday at 6pm.

1. Schedule Trigger fires weekly
2. Get row(s) in sheet fetches all rows with status = New
3. HasNew Opportunities checks if any rows returned
4. Filter Quality Matches keeps only rows where eligible = true AND fit-score ≥ 70
5. Build Email HTML formats all qualifying opportunities into an HTML email
6. Send Digest Email sends via Gmail
7. Mark As Notified updates status to Notified in parallel with email send

---

### Workflow 3: Deadline Alerts

Runs every day at 8am.

1. Schedule Trigger fires daily at 8am
2. Get Upcoming Deadline fetches all rows from Opportunities sheet
3. Filter Upcoming Deadline keeps only rows where deadline falls within the next 7 days, skips Rolling Basis and Ongoing entries
4. Send Deadline Alert sends one Gmail alert per qualifying opportunity

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow Automation | n8n |
| AI / LLM | Groq API — Llama 3.3-70B Versatile |
| Storage | Google Sheets (OAuth2) |
| Email | Gmail (OAuth2) |
| Hashing | MD5 via Node.js crypto module |
| RSS Ingestion | n8n RSS Feed Read node |

---

## Data Sources

- **OpportunityDesk** (`opportunitydesk.org/feed`) — Scholarships, fellowships, grants, accelerators
- **OpportunitiesFeed** (`opportunitiesfeed.com/feed`) — Jobs, internships, scholarships, remote careers

---

## Google Sheets Schema

**Profile sheet** — one row, columns:

`name | field | degree-level | cgpa | country | skill | opportunity-types | location-pref | resume-text`

**Opportunities sheet** — one row per opportunity:

`title | org | url | deadline | raw_description | hash | eligible | eligibility-reason | fit-score | fit-rationale | checklist | status | date added`

**Status values:** `New` / `Not Eligible` / `LowFit` / `Notified`

---

## Setup

1. Import all 3 workflow JSON files into your n8n instance
2. Set up credentials:
   - Google Sheets — OAuth2
   - Gmail — OAuth2
   - Groq API — Bearer Auth (get key at console.groq.com)
3. Create a Google Sheets file with two tabs: `Profile` and `Opportunities` using the schemas above
4. Fill in your profile row in the Profile tab
5. Activate all three workflows
6. After importing, re-link your own credentials in each node — the credential IDs in the JSON files are account-specific and will not transfer

---

## Key Design Decisions

**Three separate AI calls instead of one** — eligibility, fit scoring, and checklist generation use distinct prompts with distinct JSON schemas. Each step is independently debuggable and replaceable without touching the others.

**MD5 dedup gate** — items are hashed on normalized title + org before any AI calls are made. Duplicates are detected and dropped without wasting any API quota.

**Regex deadline extraction** — runs at Parse Eligibility time for every item regardless of eligibility outcome, so all branches (Not Eligible, LowFit, New) have deadline data stored.

**Branching storage** — three separate Google Sheets append operations handle the three outcome paths, each saving only the fields relevant to that path.

---

## Author

**Bisma Munir**

- GitHub: [github.com/Bisma474](https://github.com/Bisma474)
- Portfolio: [bisma-munir-portfolio.vercel.app](https://bisma-munir-portfolio.vercel.app)
- LinkedIn: [linkedin.com/in/bisma-m-a9055b30a](https://linkedin.com/in/bisma-m-a9055b30a)
