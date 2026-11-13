# Automated DevOps Job-Hunting Pipeline 🚀

An end-to-end automated pipeline built with **n8n** that scrapes job portals for DevOps roles, evaluates job descriptions using high-speed AI inference (**Groq**), filters opportunities based on custom criteria, logs matches to **Google Sheets**, and instantly alerts you via **Gmail** with direct application links.

---

## 🏗️ Architecture & Tech Stack

```
[Schedule Trigger] 
       │
       ▼
[Apify Scraper Actor] (Naukri Job Listings)
       │
       ▼
[JavaScript Pre-processor] (Payload Normalization & Sanitization)
       │
       ▼
[Groq AI Agent] (LLM Inference: Match Scoring & Gap Analysis)
       │
       ▼
[JavaScript Post-processor] (Filtering matchScore >= 50%)
       ├─────────────────────────┐
       ▼                         ▼
[Google Sheets]             [Gmail Alert] (HTML Email with Clickable Link)
```

* **Orchestration:** [n8n](https://n8n.io/) (Self-hosted / Cloud)
* **Web Scraping:** [Apify](https://apify.com/) (`epicscrapers~naukri-scraper` / `gio21/naukri-jobs-scraper`)
* **AI Inference:** [Groq API](https://groq.com/) (`llama-3.3-70b-versatile` / high-speed developer LLM endpoints)
* **Data Logging:** Google Sheets API
* **Notifications:** Gmail API (HTML-formatted alerts with direct application buttons)

---

## 📋 Workflow Execution Steps

1. **Trigger:** Runs on a custom schedule (e.g., daily at 9:00 AM).
2. **Scrape:** Triggers the Apify actor to fetch fresh DevOps job postings from Naukri.
3. **Pre-process:** Normalizes job payloads, handles multiple location formats/placeholders, strips HTML tags from descriptions, and limits execution batches to protect rate limits.
4. **AI Evaluation:** Passes sanitized job details into the **Groq AI Agent** with a specialized System Prompt instructing the model to act as an expert DevOps Technical Recruiter evaluating candidate fit, matching skills, missing skills, and a quantitative match score (0–100).
5. **Post-process & Filter:** Evaluates AI JSON output. Only passes jobs with a `matchScore >= 50%` downstream.
6. **Actions:** 
   * Appends candidate fit analysis and job metadata to **Google Sheets**.
   * Sends an instantaneous HTML-formatted **Gmail** notification featuring company name, title, match score, reasoning, key skills, and a clickable direct application link.

---

## 🛠️ Setup & Installation

### 1. Prerequisites
* An active [n8n](https://n8n.io/) instance.
* An [Apify](https://apify.com/) account and API token.
* A free [GroqCloud](https://console.groq.com/) account and API key.
* Google Account credentials configured in n8n for Gmail and Google Sheets nodes.

### 2. Import the Workflow
1. Download the workflow JSON file from the repository: `workflows/naukri-devops-agent.json`.
2. Open your n8n dashboard, click **Add workflow** $\rightarrow$ **Import from File**, and select the downloaded JSON.

### 3. Configure Credentials
Update the following credentials inside your n8n workspace:
* **Apify API:** Add your personal API token (`apify_api_...`) to the HTTP Request node query parameters or header auth.
* **Groq Chat Model:** Create a new credential with your Groq API key (`gsk_...`).
* **Google OAuth2:** Authorize your Gmail and Google Sheets credentials.

---

## 📊 Technical Retrospective & Troubleshooting

| Challenge | Root Cause | Solution Implemented |
| :--- | :--- | :--- |
| **API Rate Limits** | Gemini Free tier daily request cap (20 requests/day). | Migrated AI inference layer to **Groq** for high-speed, reliable, developer-friendly rate limits. |
| **Missing Metadata in Alerts** | AI Agent nodes output only generated text (`output`), dropping prior node metadata. | Implemented item-index mapping in the post-processing JavaScript node to re-attach `title`, `company`, `location`, and `applyUrl`. |
| **Unclickable Relative Links** | Scraper returned relative paths (`/job-listings-...`) instead of full URLs. | Added domain-prepending logic (`https://www.naukri.com`) and absolute URL fallbacks in the pre-processor script. |
| **Broken HTML Emails** | Gmail node defaulting to plain text formatting. | Configured Gmail node body type to **HTML** and wrapped application links in anchor tags (`<a href="..." target="_blank">`). |

---

## 🚀 Future Enhancements
* Add automated resume tailoring based on missing skills identified by Groq.
* Integrate Slack webhook notifications alongside Gmail.
