# n8n-naukri-DevOps-Job-agent
This is a project which automates applying for jobs in naukri using n8n AI Agent

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

1. **Scrapes** real-time DevOps and Cloud Engineer job listings from Naukri across specified locations (`Bangalore`, `Chennai`, `Hyderabad`, `Coimbatore`, `Kerala`).
2. **Filters & Pre-processes** raw JSON payloads into clean, lightweight evaluation blocks.
3. **Evaluates** match compatibility using **Google Gemini** against a 1+ year AWS Cloud & DevOps candidate profile.
4. **Logs** high-matching roles into a **Google Sheets** tracking table.
5. **Dispatches** real-time structured HTML alert digests directly via **Gmail**.

---

## 📐 Pipeline Architecture

The workflow follows a linear, left-to-right event-driven architecture designed to operate smoothly within free-tier API rate limits.

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

* **Endpoint:** `https://api.apify.com/v2/acts/epicscrapers~naukri-scraper/run-sync-get-dataset-items`
* **Target Roles:** DevOps, AWS Cloud Engineer, Platform Engineer.
* **Batch Sizing:** Configured with `"maxResultsPerQuery": 5` during testing to manage LLM rate limits.

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
