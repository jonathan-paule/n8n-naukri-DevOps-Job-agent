# 🚀 Autonomous AI Job Scraping & Match Evaluation Pipeline

<p align="center">
  <b>An end-to-end automated DevOps & Cloud Engineering job search orchestration pipeline built using n8n, Apify, Google Gemini AI, Google Sheets, and Gmail.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Orchestration-n8n-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white" alt="n8n" />
  <img src="https://img.shields.io/badge/Scraper-Apify-02B159?style=for-the-badge&logo=apify&logoColor=white" alt="Apify" />
  <img src="https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?style=for-the-badge&logo=googlegemini&logoColor=white" alt="Gemini" />
  <img src="https://img.shields.io/badge/Database-Google%20Sheets-34A853?style=for-the-badge&logo=googlesheets&logoColor=white" alt="Google Sheets" />
  <img src="https://img.shields.io/badge/Alerts-Gmail-EA4335?style=for-the-badge&logo=gmail&logoColor=white" alt="Gmail" />
</p>

---

## 📌 Executive Summary

Searching for target roles manually across regional tech hubs is time-consuming and inefficient. This project automates the entire candidate-to-job matching lifecycle:

1. 🔍 **Scrapes** real-time DevOps and Cloud Engineer job listings from Naukri across specified locations (`Bangalore`, `Chennai`, `Hyderabad`, `Coimbatore`, `Kerala`).
2. 🧹 **Filters & Pre-processes** raw JSON payloads into clean, lightweight evaluation blocks.
3. 🧠 **Evaluates** match compatibility using **Google Gemini** against a 1+ year AWS Cloud & DevOps candidate profile.
4. 📊 **Logs** high-matching roles into a **Google Sheets** tracking table.
5. 📬 **Dispatches** real-time structured HTML alert digests directly via **Gmail**.

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
[ ⚡ Code Node 1 (Pre-Processing & Quota Limiter) ]
          │
          ▼
[ 🤖 AI Agent Node (Google Gemini Chat Model) ]
          │
          ▼
[ 🛠️ Code Node 2 (Output Parser & Match Threshold Filter) ]
          │
          ├──► [ 📊 Google Sheets Node (Row Logger) ]
          └──► [ 📧 Gmail Node (Email Notification) ]
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
* ⚡ **Batch Sizing:** Configured with `"maxResultsPerQuery": 5` during testing to manage LLM rate limits.

<p align="center">
  <img src="assets/screenshots/02-apify-http-config.png" alt="Apify HTTP Request Configuration" width="800" />
</p>

---

### 2️⃣ JavaScript Pre-Processing & Data Normalization
A custom **Code Node (`JavaScript`)** sanitizes the raw HTML/JSON output from Naukri, handles missing fields with safe fallbacks, cleans up text formatting, and slices the payload for execution safety.

```javascript
// n8n Code Node - JavaScript Pre-processing
const items = $input.all();
const cleanedJobs = [];

// Limit to top items for quota preservation
const limitedItems = items.slice(0, 5);

for (const item of limitedItems) {
  const data = item.json;
  cleanedJobs.push({
    json: {
      jobId: data.jobId || data.id || `job_${Math.random().toString(36).substr(2, 9)}`,
      title: data.title || data.jobTitle || 'N/A',
      company: data.companyName || data.company || 'N/A',
      location: Array.isArray(data.location) ? data.location.join(', ') : (data.location || 'N/A'),
      experience: data.experience || '1-2 years',
      jdText: (data.description || data.jobDescription || data.snippet || '').replace(/<[^>]*>?/gm, '').trim(),
      applyUrl: data.url || data.jobUrl || '',
      postedDate: data.postedDate || new Date().toISOString().split('T')[0]
    }
  });
}

return cleanedJobs;
```

---

### 3️⃣ AI Match Evaluation (Google Gemini Agent)
The **AI Agent** node combines candidate specifications with job descriptions and passes them to the connected **Google Gemini Chat Model** sub-node (`models/gemini-1.5-flash` or `models/gemini-3.5-flash`).

#### 👤 Candidate Profile Prompt:
> **Role:** Technical Recruiter Evaluator  
> **Candidate Profile:** 1+ years experience in AWS Cloud & DevOps Engineering. Core skills include AWS (EC2, S3, VPC, IAM, RDS, Lambda, CloudWatch), Terraform, Docker, Kubernetes, CI/CD Pipelines (Jenkins / GitHub Actions), Python, Shell Scripting, and Linux Administration.  
> **Evaluation Criteria:** Analyze candidate alignment against the scraped `jdText`. Output strict JSON specifying `matchScore` (0–100), `seniorityFit`, `matchingSkills`, `missingSkills`, and concise `reasoning`.

---

### 4️⃣ Output Parsing & Match Filtering
The raw text returned by the AI is parsed back into JSON by **`Code in JavaScript1`**. The code applies a strict fit filter (e.g., `matchScore >= 50`) to filter out irrelevant or over-senior roles.

```javascript
// n8n Code Node - JavaScript Post-processing
const items = $input.all();
const filteredJobs = [];

for (const item of items) {
  let aiData = item.json.output;
  
  if (typeof aiData === 'string') {
    try {
      aiData = JSON.parse(aiData.replace(/```json\n?|\n?```/g, ''));
    } catch (e) {
      continue;
    }
  }

  if (aiData && aiData.matchScore >= 50) {
    filteredJobs.push({
      json: {
        matchScore: aiData.matchScore,
        seniorityFit: aiData.seniorityFit,
        matchingSkills: aiData.matchingSkills,
        missingSkills: aiData.missingSkills,
        reasoning: aiData.reasoning
      }
    });
  }
}

return filteredJobs;
```

---

### 5️⃣ Data Persistence & Alerting

* 📊 **Google Sheets Tracking:** All qualified matches are appended to a central Google Sheet job tracking table.
* 📬 **Gmail Digest Notification:** An instant email digest is dispatched to the user containing score breakdowns and key skill fits.

---

## 🛠️ Troubleshooting & Technical Retrospective

| Issue Encountered | Root Cause | Resolution Implemented |
| :--- | :--- | :--- |
| **`404 Actor Not Found`** | Incorrect Apify owner slug (`apify~naukri-scraper`) | Switched API endpoint to active author namespace (`epicscrapers~naukri-scraper`). |
| **`429 Quota Exceeded`** | Passing 40 items simultaneously into Gemini Free Tier | Added `.slice(0, 5)` in JS pre-processing and configured 15s retry waits. |
| **`404 Model Not Found`** | Deprecated model endpoint (`gemini-2.5-flash`) | Updated Gemini sub-node to stable `gemini-1.5-flash` / `gemini-3.5-flash`. |
| **`503 Service Unavailable`** | Google Cloud API capacity spikes | Enabled **Retry On Fail** (Max Tries: 5, Wait: 5000ms) in Node Settings. |
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
   * Add your **Apify API Token** in `HTTP Request1`.
   * Add your **Google Gemini API Key** in `Google Gemini Chat Model`.
   * Authorize **Google Sheets** and **Gmail** OAuth2 connections.

4. 🟢 **Activate:**
   * Toggle the **Publish** switch to enable scheduled execution.

---

<p align="center">
  <b>Built with 💻 DevOps Automation and 🤖 AI Engineering</b>
</p>
