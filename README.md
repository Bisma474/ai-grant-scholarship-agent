# n8n Opportunity Agent Workflows

An AI-powered opportunity discovery and notification system built with n8n. Automatically scrapes opportunities from multiple RSS feeds, deduplicates them, checks student eligibility using Groq AI, scores fit, and sends curated email digests.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Workflows](#workflows)
  - [1. Opportunity Agent Ingestion](#1-opportunity-agent-ingestion)
  - [2. Opportunity Agent Digest](#2-opportunity-agent-digest)
  - [3. Deadline Alerts](#3-deadline-alerts)
- [Google Sheets Structure](#google-sheets-structure)
- [Required Credentials](#required-credentials)
- [Setup Instructions](#setup-instructions)
- [How to Upload Workflow Screenshots to GitHub](#how-to-upload-pics-on-github)
- [Tech Stack](#tech-stack)

---

## Overview

This system automates the entire opportunity management lifecycle:

1. **Scrape** - Pulls opportunities from 3 RSS feeds every 6 hours
2. **Deduplicate** - MD5 hash-based deduplication against existing records
3. **Eligibility Check** - AI evaluates if a student profile matches the opportunity
4. **Fit Scoring** - AI scores how well the opportunity matches (0-100)
5. **Checklist Generation** - AI generates application checklists for high-fit opportunities
6. **Digest** - Sends weekly email digest of quality matches
7. **Deadline Alerts** - Sends email alerts for deadlines within 7 days

---

## Architecture

```
RSS Feeds (3 sources)
        |
        v
  [Merge & Normalize]
        |
        v
  [MD5 Hash + Dedup Check]
        |
        v
  [Limit to 10 items/run]
        |
        v
  [AI: Eligibility Check] -----> Not Eligible --> Log to Sheet
        |
   Eligible
        |
        v
  [AI: Fit Scoring (0-100)]
        |
        +--- Score >= 70 --> [AI: Checklist Gen] --> Save to Sheet
        |
        +--- Score < 70  --> Log as Low Fit to Sheet

  [Weekly Digest] --> Filter "New" status + Quality --> HTML Email --> Gmail
  [Deadline Alerts] --> Filter 7-day window --> HTML Email --> Gmail
```

---

## Workflows

### 1. Opportunity Agent Ingestion

**File:** `Opportunity Agent Ingestion (2).json`
**Schedule:** Every 6 hours
**Status:** Inactive (set to active after setup)

#### Node-by-Node Breakdown

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | `Execute Workflow1` | Schedule Trigger | Triggers every 6 hours |
| 2 | `Get Profile` | Google Sheets | Reads student profile from "Profile" sheet |
| 3 | `Profile Limit` | Limit | Limits to 1 profile row |
| 4 | `RSS Read` | RSS Feed | Scrapes `https://opportunitydesk.org/feed/` |
| 5 | `RSS Read1` | RSS Feed | Scrapes `https://opportunitiesforyouth.org/feed/` |
| 6 | `RSS Read2` | RSS Feed | Scrapes `https://opportunitiescorners.com/feed/` |
| 7 | `Merge1` | Merge | Combines all 3 RSS feeds into one stream |
| 8 | `Code in JavaScript` | Code | Normalizes RSS fields (title, org, url, description, date) |
| 9 | `Generate Hash` | Code | Creates MD5 hash from `title + org` for deduplication |
| 10 | `Check Duplicate` | Google Sheets | Looks up hash in "Opportunities" sheet |
| 11 | `Dedup Check` | Code | Passes only new (non-duplicate) items |
| 12 | `Limit Items` | Limit | Caps at 10 items per run (API cost control) |
| 13 | `Build Eligibility Body` | Code | Constructs AI prompt with student profile + opportunity |
| 14 | `Check Eligibility` | HTTP Request | POST to Groq API (`openai/gpt-oss-20b`) for eligibility |
| 15 | `Parse Eligibility` | Code | Parses AI response, extracts deadline from description |
| 16 | `Is Eligible` | If | Routes based on `eligible: true/false` |
| 17 | `Append row in sheet` | Google Sheets | Logs ineligible opportunities |
| 18 | `Build Score Fit Body` | Code | Constructs fit-scoring AI prompt |
| 19 | `Score Fit` | HTTP Request | POST to Groq API for fit score (0-100) |
| 20 | `Parse Fit Score` | Code | Parses score and rationale from AI response |
| 21 | `Fit Threshold` | If | Routes based on `fitScore >= 70` |
| 22 | `Build Checklist Body` | Code | Constructs checklist-generation AI prompt |
| 23 | `Generate Checklist` | HTTP Request | POST to Groq API for application checklist |
| 24 | `Parse Checklist` | Code | Parses checklist and resume gaps |
| 25 | `Save Opportunity` | Google Sheets | Saves high-fit opportunities to "Opportunities" sheet |
| 26 | `Low Fit` | Google Sheets | Logs low-fit opportunities separately |

#### Data Flow Details

**Step 1 - Profile Loading:**
The workflow reads the student's profile from the "Profile" sheet containing:
- `field` - Academic field
- `degree-level` - Current degree level
- `cgpa` - GPA
- `country` - Location
- `skill` - Skills list
- `resume-text` - Resume summary

**Step 2 - RSS Scraping:**
Three RSS feeds are scraped in parallel:
- OpportunityDesk.org
- OpportunitiesForYouth.org
- OpportunitiesCorners.com

**Step 3 - Deduplication:**
- MD5 hash is generated from `title + org` (lowercased, whitespace-normalized)
- Hash is checked against existing records in Google Sheets
- Only new items pass through

**Step 4 - AI Eligibility Check:**
The Groq API (`openai/gpt-oss-20b`) evaluates:
```
Student Profile (field, degree, CGPA, country, skills)
        +
Opportunity (title, org, description)
        =
{ "eligible": true/false, "reason": "...", "missing_requirements": [] }
```

**Step 5 - AI Fit Scoring:**
For eligible opportunities, AI scores 0-100:
```
Student Profile (field, skills, resume) + Opportunity = Fit Score + Rationale
```

**Step 6 - AI Checklist Generation:**
For fit score >= 70, AI generates:
- Required documents list
- Internal deadline pacing plan
- Resume gap analysis

**Step 7 - Storage:**
Results are saved to the "Opportunities" sheet with columns:
`title`, `org`, `url`, `deadline`, `raw_description`, `hash`, `eligible`, `eligibility-reason`, `fit-score`, `fit-rationale`, `checklist`, `status`, `date added`

---

### 2. Opportunity Agent Digest

**File:** `Opportunity Agent-Digest (1).json`
**Schedule:** Every hour (checks for new quality matches)
**Status:** Active

#### Node-by-Node Breakdown

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | `Schedule Trigger` | Schedule Trigger | Runs hourly |
| 2 | `Get row(s) in sheet` | Google Sheets | Fetches rows where `status = "New"` |
| 3 | `HasNew Opportunities` | If | Checks if any rows have a non-empty title |
| 4 | `Filter Quality Matches` | Filter | Keeps items where `eligible = true` OR `fit-score >= 70` |
| 5 | `Build Email HTML` | Code | Generates styled HTML email with all matching opportunities |
| 6 | `Send Digest Email` | Gmail | Sends digest to `bismamunir474@gmail.com` |
| 7 | `Mark As Notified` | Google Sheets | Updates `status` from "New" to "Notified" |

#### Email Format

The digest email includes for each opportunity:
- Title and fit score (e.g., "Scholarship Program (Fit: 85/100)")
- Organization name
- AI-generated rationale
- Application checklist
- Link to the opportunity

#### Status Flow

```
"New" --> [Digest Sent] --> "Notified"
```

---

### 3. Deadline Alerts

**File:** `Deadline alerts (1).json`
**Schedule:** Every 4 hours
**Status:** Active

#### Node-by-Node Breakdown

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | `Schedule Trigger` | Schedule Trigger | Runs every 4 hours |
| 2 | `Get Upcoming Deadline` | Google Sheets | Reads all rows from "Opportunities" sheet |
| 3 | `Filter Upcoming Deadline` | Code | Filters deadlines within the next 7 days (excludes "rolling" and "ongoing") |
| 4 | `Send Deadline Alert` | Gmail | Sends HTML email alert per deadline |

#### Alert Email Format

```html
⏰ Deadline Approaching!
Title: [Opportunity Title]
Org: [Organization]
Deadline: [Date]
Status: [Status]
View Opportunity: [Link]
```

---

## Google Sheets Structure

### Spreadsheet: "Opportunity agent"

**Sheet: Profile** (gid=0)
| Column | Description |
|--------|-------------|
| field | Academic field of study |
| degree-level | Current degree (e.g., Bachelor's, Master's) |
| cgpa | GPA value |
| country | Student's country |
| skill | Comma-separated skills |
| resume-text | Resume summary text |

**Sheet: Opportunities** (gid=2014962491)
| Column | Description |
|--------|-------------|
| title | Opportunity title |
| org | Organization name |
| url | Link to opportunity |
| deadline | Application deadline |
| raw_description | Full opportunity description |
| hash | MD5 hash for deduplication |
| eligible | AI eligibility result (true/false) |
| eligibility-reason | AI explanation for eligibility |
| fit-score | AI fit score (0-100) |
| fit-rationale | AI explanation for score |
| checklist | AI-generated application checklist |
| status | "New" or "Notified" |
| date added | Date the opportunity was ingested |

---

## Required Credentials

| Credential | Used By | Purpose |
|------------|---------|---------|
| `Google Sheets OAuth2 API` | All workflows | Read/write Google Sheets |
| `Gmail OAuth2` | Digest + Deadline alerts | Send email notifications |
| `Groq API` | Ingestion | AI eligibility checks and scoring |

### Setting Up Credentials in n8n

1. **Google Sheets OAuth2:**
   - Go to Google Cloud Console > APIs & Services > Credentials
   - Create OAuth 2.0 Client ID (Desktop app)
   - Enable Google Sheets API
   - In n8n: Settings > Credentials > Add "Google Sheets OAuth2 API"
   - Paste Client ID and Client Secret
   - Authenticate via browser

2. **Gmail OAuth2:**
   - Same Google Cloud project as above
   - Enable Gmail API
   - In n8n: Add "Gmail OAuth2" credential
   - Authenticate and grant send permission

3. **Groq API:**
   - Sign up at https://console.groq.com
   - Generate API key
   - In n8n: Add "Groq API" credential
   - Paste API key

---

## Setup Instructions

### Prerequisites

- n8n instance (self-hosted or cloud)
- Google Cloud project with Sheets + Gmail APIs enabled
- Groq API account and key
- A Google Spreadsheet set up with "Profile" and "Opportunities" sheets

### Steps

1. **Import Workflows:**
   - Open n8n dashboard
   - Click "Import from File" (or paste JSON)
   - Import each `.json` file:
     - `Opportunity Agent Ingestion (2).json`
     - `Opportunity Agent-Digest (1).json`
     - `Deadline alerts (1).json`

2. **Configure Credentials:**
   - Click on each node that has a credential (Google Sheets, Gmail, Groq)
   - Select or create the corresponding credential
   - Test the connection

3. **Update Google Sheet References:**
   - The workflows reference a specific spreadsheet ID: `18idCeo7nO_iBMn0La5FIYknqKzv02XwrCgFg2aTbnbk`
   - Replace this with your own spreadsheet ID in all Google Sheets nodes
   - Ensure sheet names match: "Profile" and "Opportunities"

4. **Update RSS Feed URLs (Optional):**
   - Modify the RSS Read nodes if you want different sources
   - Current feeds:
     - `https://opportunitydesk.org/feed/`
     - `https://opportunitiesforyouth.org/feed/`
     - `https://opportunitiescorners.com/feed/`

5. **Update Email Address:**
   - Change `bismamunir474@gmail.com` in the Gmail nodes to your email

6. **Fill Student Profile:**
   - Go to your Google Sheet > "Profile" tab
   - Fill in your academic details, skills, and resume summary

7. **Activate Workflows:**
   - Toggle each workflow to "Active" in the n8n dashboard

### Workflow Execution Order

```
1. Opportunity Agent Ingestion  (every 6 hours)  --> Saves to Sheet
2. Opportunity Agent Digest     (every 1 hour)   --> Sends email digest
3. Deadline Alerts              (every 4 hours)  --> Sends deadline emails
```

---

## How to Upload Pics on GitHub

If you want to add workflow screenshots or diagrams to this repository:

### Method 1: Direct Upload (via GitHub Web UI)

1. Go to your repository on GitHub
2. Click **"Add file"** > **"Upload files"**
3. Drag and drop your images (PNG, JPG, GIF, SVG)
4. Commit the changes

### Method 2: Git Command Line

```bash
# Copy images into your repo folder
cp /path/to/screenshot.png ./images/

# Stage and commit
git add images/screenshot.png
git commit -m "Add workflow screenshot"
git push origin main
```

### Method 3: Reference Images in README

After uploading, reference them in your README:

```markdown
![Workflow Diagram](images/workflow-diagram.png)

<!-- With custom size -->
<img src="images/screenshot.png" width="600">

<!-- As a clickable link -->
[![Workflow](images/thumb.png)](images/full-screenshot.png)
```

### Recommended Image Workflow

```
1. Take screenshots of each workflow in n8n editor
2. Create an "images/" folder in your repo
3. Name them clearly:
   - images/ingestion-workflow.png
   - images/digest-workflow.png
   - images/deadline-workflow.png
   - images/architecture-diagram.png
4. Upload via GitHub web UI or git
5. Reference in README.md
```

### GitHub Image Size Limits

- **Max file size:** 25 MB per image (but keep under 1 MB for fast loading)
- **Recommended formats:** PNG for screenshots, SVG for diagrams
- **Recommended width:** 800-1200px for README images

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Workflow Engine | n8n |
| AI Model | Groq API (openai/gpt-oss-20b) |
| Data Store | Google Sheets |
| Notifications | Gmail (OAuth2) |
| RSS Sources | OpportunityDesk, OpportunitiesForYouth, OpportunitiesCorners |
| Deduplication | MD5 Hashing |
| Scheduling | n8n Schedule Trigger nodes |

---

## License
