# 🚀 Autonomous AI Job Scraping & Match Evaluation Pipeline

<p align="center">
  <b>An end-to-end automated DevOps & Cloud Engineering job search orchestration pipeline built using n8n, Apify, Groq AI, Google Sheets, and Gmail.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Orchestration-n8n-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white" alt="n8n" />
  <img src="https://img.shields.io/badge/Scraper-Apify-02B159?style=for-the-badge&logo=apify&logoColor=white" alt="Apify" />
  <img src="https://img.shields.io/badge/LLM-Groq%20AI-F55036?style=for-the-badge&logo=groq&logoColor=white" alt="Groq AI" />
  <img src="https://img.shields.io/badge/Database-Google%20Sheets-34A853?style=for-the-badge&logo=googlesheets&logoColor=white" alt="Google Sheets" />
  <img src="https://img.shields.io/badge/Alerts-Gmail-EA4335?style=for-the-badge&logo=gmail&logoColor=white" alt="Gmail" />
</p>

---

## 📌 Executive Summary

Searching for target roles manually across regional tech hubs is time-consuming and inefficient. This project automates the entire candidate-to-job matching lifecycle:

1. 🔍 **Scrapes** real-time DevOps and Cloud Engineer job listings from Naukri across specified locations (`Bangalore`, `Chennai`, `Hyderabad`, `Coimbatore`, `Kerala`).
2. 🧹 **Filters & Pre-processes** raw JSON payloads into clean, normalized evaluation blocks with absolute Naukri application links.
3. 🧠 **Evaluates** match compatibility using **Groq AI** against a 1+ year AWS Cloud & DevOps candidate profile.
4. 📊 **Logs** high-matching roles into a **Google Sheets** tracking table.
5. 📬 **Dispatches** real-time structured HTML alert digests with clickable application links directly via **Gmail**.

---

## 📐 Pipeline Architecture

The workflow follows a linear, left-to-right event-driven architecture designed to operate smoothly within free-tier API rate limits:

```text
[ ⏰ Schedule Trigger ]
          │
          ▼
[ 🌐 HTTP Request (Apify Scraper) ]
          │
          ▼
[ ⚡ Code Node 1 (Pre-Processing & URL Normalization) ]
          │
          ▼
[ 🤖 AI Agent Node (Groq Chat Model) ]
          │
          ▼
[ 🛠️ Code Node 2 (Output Parser, Metadata Merger & Threshold Filter) ]
          │
          ├──► [ 📊 Google Sheets Node (Row Logger) ]
          └──► [ 📧 Gmail Node (HTML Email Notification) ]
```

---

## 📸 Workflow Execution Overview

<p align="center">
  <img src="assets/screenshots/01-canvas-overview.png" alt="n8n Complete Pipeline Canvas" width="900" />
  <br>
  <i>Figure 1: Complete end-to-end n8n workflow canvas showing green active status across all execution nodes.</i>
</p>

---

## ⚙️ Step-by-Step Implementation Guide

### 1️⃣ Automated Data Collection (Apify HTTP Request)
The pipeline is triggered automatically via a **Schedule Trigger** node. It sends a `POST` payload to the Apify REST API to execute the `epicscrapers~naukri-scraper` Actor.

* 🎯 **Endpoint:** `https://api.apify.com/v2/acts/epicscrapers~naukri-scraper/run-sync-get-dataset-items`
* 💼 **Target Roles:** DevOps, AWS Cloud Engineer, Platform Engineer.
* ⚡ **Batch Sizing:** Configured with `"maxResultsPerQuery": 5` during execution to optimize speed and API quotas.

<p align="center">
  <img src="assets/screenshots/02-apify-http-config.png" alt="Apify HTTP Request Configuration" width="800" />
</p>

---

### 2️⃣ JavaScript Pre-Processing & Data Normalization
A custom **Code Node (`Code in JavaScript`)** sanitizes the raw HTML/JSON output from Naukri, extracts nested location placeholders, prepends `https://www.naukri.com` to relative paths so application links are clickable, and slices the payload for execution safety.

```javascript
// n8n Code Node - JavaScript (Pre-processing)
const items = $input.all();
const cleanedJobs = [];

// Limit to the first 5 jobs to protect API quotas during execution
const limitedItems = items.slice(0, 5);

for (const item of limitedItems) {
  const data = item.json;
  
  // Extract location with multi-source fallbacks (handles arrays, strings, and object placeholders)
  let resolvedLocation = 'N/A';
  if (Array.isArray(data.location)) {
    resolvedLocation = data.location.join(', ');
  } else if (typeof data.location === 'string' && data.location.trim() !== '') {
    resolvedLocation = data.location;
  } else if (Array.isArray(data.placeholders)) {
    const locPlaceholder = data.placeholders.find(p => p.type === 'location');
    if (locPlaceholder && locPlaceholder.label) {
      resolvedLocation = locPlaceholder.label;
    }
  }

  // Extract application URL and ensure it's an absolute URL
  let resolvedUrl = data.jdURL || data.url || data.jobUrl || data.staticUrl || '';
  if (resolvedUrl.startsWith('/')) {
    resolvedUrl = `https://www.naukri.com` + resolvedUrl;
  } else if (!resolvedUrl && data.jobId) {
    resolvedUrl = `https://www.naukri.com/job-listings-${data.jobId}`;
  }

  // Extract and clean job description text
  const rawDesc = data.description || data.jobDescription || data.snippet || '';
  const cleanedDesc = typeof rawDesc === 'string' 
    ? rawDesc.replace(/<[^>]*>?/gm, '').trim() 
    : '';

  // Construct normalized output item
  cleanedJobs.push({
    json: {
      jobId: data.jobId || data.id || `job_${Math.random().toString(36).substr(2, 9)}`,
      title: data.title || data.jobTitle || 'N/A',
      company: data.companyName || data.company || 'N/A',
      location: resolvedLocation,
      experience: data.experience || '1-2 years',
      jdText: cleanedDesc,
      applyUrl: resolvedUrl,
      postedDate: data.postedDate || new Date().toISOString().split('T')[0]
    }
  });
}

return cleanedJobs;
```

---

### 3️⃣ AI Match Evaluation (Groq AI Agent)
The **AI Agent** node combines candidate specifications with job descriptions and passes them to the connected **Groq Chat Model** sub-node (`openai/gpt-oss-20b` / `openai/gpt-oss-120b`).

<p align="center">
  <img src="assets/screenshots/04-groq-ai-agent.png" alt="Groq AI Agent Configuration" width="800" />
</p>

#### 👤 Candidate Profile Prompt:
> **Role:** Technical Recruiter Evaluator  
> **Candidate Profile:** 1+ years experience in AWS Cloud & DevOps Engineering. Core skills include AWS (EC2, S3, VPC, IAM, RDS, Lambda, CloudWatch), Terraform, Docker, Kubernetes, CI/CD Pipelines (Jenkins / GitHub Actions), Python, Shell Scripting, and Linux Administration.  
> **Evaluation Criteria:** Analyze candidate alignment against the scraped `jdText`. Output strict JSON specifying `matchScore` (0–100), `seniorityFit`, `matchingSkills`, `missingSkills`, and concise `reasoning`.

---

### 4️⃣ Output Parsing, Metadata Merging & Match Filtering
Because n8n's AI Agent node only outputs its generated response (`output`), **`Code in JavaScript1`** parses the JSON evaluation, merges the original job metadata (`title`, `company`, `location`, `applyUrl`) back from `Code in JavaScript` via index mapping, and filters out roles scoring below `50%`.

```javascript
// n8n Code Node - JavaScript Post-processing (with original job metadata)
const aiItems = $input.all();
const originalJobs = $('Code in JavaScript').all();
const filteredJobs = [];

for (let i = 0; i < aiItems.length; i++) {
  let aiData = aiItems[i].json.output;
  const orig = originalJobs[i] ? originalJobs[i].json : {};

  if (typeof aiData === 'string') {
    try {
      aiData = JSON.parse(aiData.replace(/```json\n?|\n?```/g, '').trim());
    } catch (e) {
      continue;
    }
  }

  if (aiData && aiData.matchScore >= 50) {
    filteredJobs.push({
      json: {
        jobId: orig.jobId || 'N/A',
        title: orig.title || 'N/A',
        company: orig.company || 'N/A',
        location: orig.location || 'N/A',
        applyUrl: orig.applyUrl || 'N/A',
        postedDate: orig.postedDate || '',
        matchScore: aiData.matchScore,
        seniorityFit: aiData.seniorityFit,
        matchingSkills: Array.isArray(aiData.matchingSkills) ? aiData.matchingSkills.join(', ') : aiData.matchingSkills,
        missingSkills: Array.isArray(aiData.missingSkills) ? aiData.missingSkills.join(', ') : aiData.missingSkills,
        reasoning: aiData.reasoning
      }
    });
  }
}

return filteredJobs;
```

---

### 5️⃣ Data Persistence & HTML Alerting

* 📊 **Google Sheets Tracking:** All qualified matches are appended to a central Google Sheet job tracking table.
* 📬 **Gmail HTML Digest Notification:** An instant HTML-formatted email digest is dispatched to the user containing score breakdowns, matched skills, and a clickable **Apply Here** anchor link.

---

## 🛠️ Troubleshooting & Technical Retrospective

| Issue Encountered | Root Cause | Resolution Implemented |
| :--- | :--- | :--- |
| **`404 Actor Not Found`** | Incorrect Apify owner slug (`apify~naukri-scraper`) | Switched API endpoint to active author namespace (`epicscrapers~naukri-scraper`). |
| **`429 Quota Exceeded` / `503`** | Gemini Free Tier daily request limits and capacity spikes | Migrated LLM inference layer to **Groq AI** (`openai/gpt-oss-20b`) and sliced batches to top 5 items. |
| **Decommissioned Model Error** | Legacy model ID (`llama3-8b-8192`) retired on Groq | Updated Groq Chat Model sub-node to active `openai/gpt-oss-20b` endpoint. |
| **Blank Fields in Gmail Alerts** | AI Agent node dropped upstream job metadata (`title`, `company`, `applyUrl`) | Added index-based metadata merging from `$('Code in JavaScript')` inside post-processing. |
| **Unclickable Relative Links** | Scraper returned relative paths (`/job-listings-...`) and plain text email | Prepended `https://www.naukri.com` in pre-processing and switched Gmail node to **HTML** mode with `<a>` tags. |
| **`Forbidden Gmail API`** | Unenabled API / missing OAuth scopes | Enabled Gmail API in Google Cloud Console project and re-authorized n8n credentials. |

---

## 🚀 Getting Started & Setup

1. 📥 **Clone the Repository:**
   ```bash
   git clone https://github.com/your-username/n8n-naukri-devops-job-agent.git
   cd n8n-naukri-devops-job-agent
   ```

2. ⚙️ **Import Workflow:**
   * Open your local or cloud n8n instance.
   * Select **Workflows** ➔ **Import from File**.
   * Choose `workflows/naukri-devops-agent.json`.

3. 🔑 **Configure Credentials:**
   * Add your **Apify Personal API Token** (`apify_api_...`) in `HTTP Request1`.
   * Add your **Groq API Key** (`gsk_...`) in `Groq Chat Model`.
   * Authorize **Google Sheets** and **Gmail** OAuth2 connections.

4. 🟢 **Activate:**
   * Toggle the **Publish** switch to enable scheduled execution.

---

<p align="center">
  <b>Built with 💻 DevOps Automation and 🤖 AI Engineering</b>
</p>
