## AI Candidate Evaluator

This project is an end-to-end agent that evaluates job applications sent over email. It reads incoming applications, extracts signals from resumes, GitHub, and portfolios, scores candidates against a rubric, and responds with a decision.

Built to work in production and be tested live

---

## What this does

Candidates apply by emailing:

- Resume (PDF)
- GitHub link
- Portfolio link

The system:

- Reads the email
- Extracts key signals (resume, GitHub, portfolio)
- Evaluates the candidate using an AI model + rubric
- Stores the result
- Responds with either:
  - Pass (next steps)
  - Fail (with specific feedback)
- Handles edge cases (missing info, duplicates, auto-replies)

---

## Architecture

```text
Gmail Inbox  
    │  
    ▼  

┌─────────────────────────────┐  
│ Gmail Trigger               │  Polls for new application emails (every minute)  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ Parse Email & Extract       │  Extracts: name, email, GitHub URL,  
│ Signals                     │  portfolio URL, PDF resume, LinkedIn  
└────────────┬────────────────┘  
             │  
             ├── Auto-reply detected ─────► Skip silently  
             │  
             ▼  

┌─────────────────────────────┐  
│ Lookup Prior Evaluations    │  Checks Google Sheets for duplicates  
│ (Google Sheets)             │  
└────────────┬────────────────┘  
             │  
             ├── Already evaluated ───────► Log duplicate (Google Docs), stop  
             │  
             ▼  

┌─────────────────────────────┐  
│ Tag Email as Read           │  Marks thread as processed  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ IF: Application Complete?   │  Requires:  
│                             │  - PDF resume  
│                             │  - GitHub OR portfolio  
└────────────┬────────────────┘  
             │  
             ├── Incomplete ─────────────► Send missing-info reply  
             │                             (resume missing / link missing / both)  
             │  
             ▼  

┌─────────────────────────────┐  
│ Resume Processing           │  - Upload to Google Drive  
│                             │  - Extract text from PDF  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ IF: Has Portfolio?          │  
└────────────┬────────────────┘  
             │  
             ├── Yes ─────────► Fetch Portfolio Content  
             │                  Parse → clean text  
             │  
             ▼  

┌─────────────────────────────┐  
│ IF: Has GitHub?             │  
└────────────┬────────────────┘  
             │  
             ├── Yes ─────────► Fetch GitHub Profile  
             │                  Fetch GitHub Repos (recent activity)  
             │  
             ▼  

┌─────────────────────────────┐  
│ Build Evaluation Context    │  Combines: resume, GitHub, portfolio,  
│                             │  metadata, and evaluation flags  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ Call Claude API             │  Applies scoring rubric and returns  
│ (Evaluation)                │  structured JSON output  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ Parse Evaluation            │  Extracts: scores, decision, confidence,  
│                             │  strengths, concerns, inconsistencies  
└────────────┬────────────────┘  
             │  
             ├── FAIL ─────────► Generate personalised rejection  
             │                   (Claude second pass)  
             │  
             ▼  

┌─────────────────────────────┐  
│ Append to Google Sheets     │  Stores evaluation + reasoning + links  
└────────────┬────────────────┘  
             │  
             ▼  

┌─────────────────────────────┐  
│ Send Final Email            │  
│ (Gmail)                     │  PASS → next steps  
│                             │  FAIL → specific feedback  
└─────────────────────────────┘  
```

---

## Tech Stack

### Core
- n8n → workflow orchestration (main engine)  
- JavaScript (Code nodes) → custom logic + transformations  

### Integrations
- Gmail API → inbound + outbound communication  
- Google Drive → store resumes  
- Google Sheets → evaluation dashboard  
- GitHub API → candidate signal enrichment  

### AI
- Claude (Anthropic API) → evaluation + reasoning + email feedback  

---

## Why this stack

### 1. n8n (core decision)
Fastest way to build an end-to-end working system  
Built-in integrations (Gmail, Sheets, APIs)  
Great visibility into logs and execution (important for live testing)  
Easy for debugging  
One click deploy  

### 2. JavaScript inside n8n
Used for:
- Parsing emails  
- Extracting links  
- Building evaluation context  

Keeps everything flexible without over-engineering  

### 3. Claude for evaluation
Strong reasoning for structured evaluation  
Handles messy inputs well (resumes, portfolios, GitHub)  

### 4. Google Sheets as storage
Simple, transparent dashboard  
Easy to inspect during live testing  
No overhead of setting up a database  

---

## Trade-offs

### 1. n8n vs custom backend

**Why:**  
Used n8n to quickly build an end-to-end working system with minimal setup. It handles orchestration, integrations, and execution visibility out of the box.  

**Trade-off:**  
Less control over system design and scalability. Logic is distributed across nodes instead of clean, testable services. Harder to extend into a large production system.  

---

### 2. Google Sheets vs database

**Why:**  
Used Google Sheets as a lightweight, transparent dashboard for storing evaluations. Easy to inspect during live testing and requires no setup.  

**Trade-off:**  
Not scalable or structured. Limited querying, weak data modeling, and not suitable for more complex product features or analytics.  

---

### 3. Google Docs for logging vs structured logging system

**Why:**  
Used Google Docs to capture workflow-level logs in a human-readable format. Makes it easy to debug and explain behavior during live runs.  

**Trade-off:**  
Logs are unstructured and not queryable. Not suitable for monitoring, alerting, or debugging at scale.  

---

### 4. Model choice (Claude Sonnet)

**Why:**  
Used Claude Sonnet as a cost-effective model that performs well on reasoning-heavy tasks like candidate evaluation and structured output generation.  

**Trade-off:**  
Slightly lower ceiling compared to larger models. Requires careful prompting to maintain consistency across evaluations.  

---

### 5. Input requirements (Resume + either GitHub or Portfolio)

**Why:**  
Made a product decision to require a resume and at least one of GitHub or Portfolio. This accommodates different types of builders — some are code-heavy, others are product-focused.  

**Trade-off:**  
Evaluation can be less confident when one signal is missing. Requires additional handling in logic to avoid unfair penalization.  \



## Setup

This workflow runs on n8n Cloud. To run your own instance:

1. **Import the workflow.** In n8n, go to Workflows → Import from File and upload `workflow/Plum_Candidate_Evaluator.json`.
2. **Connect credentials.** Wire up four integrations: Gmail OAuth2 (for the trigger and outbound replies), Google Drive OAuth2 (resume storage), Google Sheets OAuth2 (the dashboard), and an HTTP Header Auth credential containing your Anthropic API key (`x-api-key: sk-ant-...`) for the two Claude API call nodes.
3. **Set up the Sheet and Doc.** Create a Google Sheet with a `Dashboard` tab and these column headers: `Timestamp`, `Applicant Name`, `Applicant Email`, `Status`, `Resume Link`,`Decision`, `Confidence`,`Shipped Production Products`, `Business Thinking`, `Technical Depth`, `Speed of Execution`, `Weighted Score`, `Resume Link`, `GitHub URL`, `Portfolio URL`,  `Agent Reasoning`, `Inconsistencies/Concerns`. Create a Google Doc for duplicate logging. Update the document IDs in the `Lookup Prior Evaluations`, `Append 'Evaluated' Row`, and `Log Duplicate to Doc` nodes.
4. **Activate the workflow.** Toggle the workflow active in the top-right of the n8n editor. The Gmail trigger polls every minute.
5. **Send a test email** to the connected Gmail address with subject `Application for AI Builder`, a PDF resume attached, and a GitHub or portfolio link in the body.

---

## How to test in production

The agent is live. To send a real application:

**Email:** `amoghvignesh29@gmail.com`
**Subject:** `Application for AI Builder`

Include in the email:

- Your resume as a PDF attachment
- A GitHub profile link
- A portfolio or project link

The agent polls the inbox every minute, evaluates the application, and replies within a few minutes with either a PASS (with next steps) or a FAIL (with specific feedback).

**Note:** This assumes the role is publicly advertised (e.g., a LinkedIn post stating "We're hiring an AI Builder — apply to amoghvignesh29@gmail.com with subject 'Application for AI Builder' "). The trigger filters on subject containing `Application for AI Builder`.

---

## Evaluation Framework

The evaluator scores candidates across four parameters:

- **Shipped Production Products (35%)** - evidence of building and launching real products or systems  
- **Business Thinking (30%)** - ability to understand users, trade-offs, and product impact  
- **Technical Depth (25%)** - strength of technical execution based on resume, GitHub, and projects  
- **Speed of Execution (10%)** - signals of iteration speed, recent activity, and ability to ship quickly

The final decision is based on both weighted score and confidence. The system uses a minimum weighted score of **3.5/5** and a minimum confidence threshold of **60%** to advance a candidate. This helps avoid over-advancing candidates when the signal is weak, incomplete, or inconsistent.

## Test cases

1. **Strong builder, complete application** — evaluated end-to-end and PASSed.
2. **Inflated resume vs. thin GitHub** — agent flagged the inconsistency in its reasoning and FAILed. 
3. **Missing resume** — agent replied in-thread asking for it; candidate's follow-up was picked up and evaluated on the next poll.
4. **Duplicate application** — second send was caught and skipped silently; no double-reply.
5. **Prompt injection in email body** — untrusted-data delimiters in the evaluation prompt held; agent evaluated on actual signals, not the injected instruction.

All five behaved as designed during testing
