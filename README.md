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
