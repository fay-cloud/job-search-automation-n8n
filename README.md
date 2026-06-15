# job-search-automation-n8n

An automated job search pipeline built with n8n that scrapes Data Engineer listings from Indeed Netherlands, scores each role against a candidate profile using an LLM, generates tailored cover letters for qualifying roles, and stores everything in a Google Sheets tracker. Runs on a daily schedule with no manual intervention.

---

## How it works

The workflow runs every day at 10:00 AM and executes the following steps in sequence:

1. **Schedule Trigger** fires at 10:00 daily
2. **Apify Actor** scrapes Indeed Netherlands for "Data Engineer" listings
3. **Dataset retrieval** fetches the raw results from Apify's dataset API
4. **Limit node** caps the number of jobs processed per run
5. **Resume Assessor** sends each job to GPT for structured scoring against the candidate profile, returning JSON with a score out of 10 and reasoning
6. **Cover Letter Writer** generates a tailored, ATS-aware cover letter for each job using the job description and candidate profile
7. **Google Sheets** appends or updates a row in the tracker with all job details, the score, and the generated cover letter

---

## Scoring logic

The LLM scores each job out of 10 using the following criteria:

| Criterion | Points |
|---|---|
| Excellent resume fit | 4 |
| Partial resume fit | 2 |
| Visa sponsorship offered | 3 |
| Onsite or hybrid | 1 |
| Benefits listed | 1 |
| Full-time employment | 1 |
| Dutch language required | 0 (auto-disqualify) |

Jobs that require Dutch language proficiency are automatically assigned a score of 0 regardless of other criteria.

---

## Tech stack

| Tool | Role |
|---|---|
| n8n | Workflow orchestration and scheduling |
| Apify (Indeed Scraper) | Job listing data collection |
| OpenAI GPT (gpt-5-mini) | Resume scoring and cover letter generation |
| Google Sheets | Job tracking and output storage |

---

## Workflow file

The exported n8n workflow JSON is located at:

```
workflows/job_search_automation.json
```

To import it into your n8n instance, go to Workflows > Import from file and select the JSON.

---

## Google Sheets schema

The tracker sheet uses the following columns:

| Column | Source |
|---|---|
| Title | Job listing title |
| Company Name | Scraped from Indeed |
| City | Scraped from Indeed |
| Location (Hybrid/Onsite) | `isRemote` field from listing |
| Salary | Scraped if available |
| Date | Date published |
| Platform | Hardcoded as "Indeed" |
| Link | Direct job URL |
| Job Description | Full description text |
| Rating | LLM score out of 10 |
| Cover Letter | Generated cover letter text |
| Expired | Expiry flag from listing |
| Status | Set to "ToDo" on creation |

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Apify account with access to the Indeed Jobs Scraper actor (`MXLpngmVpE8WTESQr`)
- OpenAI API key
- Google account with a Google Sheets spreadsheet created

### Credentials required in n8n

| Credential | Used by |
|---|---|
| Apify OAuth2 API | Run an Actor, Get dataset items |
| OpenAI API | Resume Assessor, Cover Letter Writer |
| Google Sheets OAuth2 API | Append or update row in sheet |

### Steps

1. Clone this repository
2. Open your n8n instance
3. Go to **Workflows > Import from file** and upload `workflows/job_search_automation.json`
4. Configure the three credentials listed above in n8n's credential manager
5. Update the Google Sheets node with your own spreadsheet ID and sheet name
6. Update the candidate profile sections inside both OpenAI nodes with your own resume content
7. Activate the workflow

---

## Configuration you need to change

Before running the workflow, update the following:

**In the `Resume Assessor` node:**
Replace the candidate profile block in the user message with your own resume content.

**In the `Cover_letter_writer` node:**
Replace the candidate profile block in the user message with your own resume content.

**In the `Append or update row in sheet` node:**
Replace the `documentId` value with the ID of your own Google Sheet. The sheet ID is the long string in the sheet URL between `/d/` and `/edit`.

**In the `Run an Actor` node:**
You can change the `query` field (currently `"Data Engineer"`) and `country` field (currently `"nl"`) to match your own job search.

---

## Project structure

```
job-search-automation-n8n/
├── workflows/
│   └── job_search_automation.json   # n8n workflow export
├── docs/
│   └── workflow_diagram.png         # Visual diagram of the pipeline
├── .gitignore
└── README.md
```

---

## Limitations and known gaps

- The `Limit` node is set to 1 job per run in the current configuration. Increase this value to process more listings per day.
- `enableUniqueJobs` is set to false in the Apify actor call, which means duplicate listings may appear across runs. Deduplication is currently handled at the Google Sheets level using the job URL as a matching key.
- The workflow does not currently filter by minimum score before generating a cover letter. All jobs that pass the Limit node receive a cover letter regardless of score. Adding an IF node between the Resume Assessor and Cover Letter Writer to filter on score >= threshold would reduce unnecessary LLM calls.
- Cover letter quality depends on the richness of the job description returned by Apify. Short or poorly structured listings will produce weaker output.

---

## License

MIT
