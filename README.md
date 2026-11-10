🚀 Autonomous AI Job Scraping & Match Evaluation Pipeline📌 Executive SummarySearching for target roles manually across regional tech hubs is time-consuming and inefficient. This project automates the entire candidate-to-job matching lifecycle:🔍 Scrapes real-time DevOps and Cloud Engineer job listings from Naukri across specified locations (Bangalore, Chennai, Hyderabad, Coimbatore, Kerala).🧹 Filters & Pre-processes raw JSON payloads into clean, lightweight evaluation blocks.🧠 Evaluates match compatibility using Google Gemini against a 1+ year AWS Cloud & DevOps candidate profile.📊 Logs high-matching roles into a Google Sheets tracking table.📬 Dispatches real-time structured HTML alert digests directly via Gmail.📐 Pipeline ArchitectureThe workflow follows a linear, left-to-right event-driven architecture designed to operate smoothly within free-tier API rate limits:[ ⏰ Schedule Trigger ]
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
📸 Workflow Execution Overview⚙️ Step-by-Step Implementation Guide1️⃣ Automated Data Collection (Apify HTTP Request)The pipeline is triggered automatically via a Schedule Trigger node. It sends a POST payload to the Apify REST API to execute the epicscrapers~naukri-scraper Actor.🎯 Endpoint: https://api.apify.com/v2/acts/epicscrapers~naukri-scraper/run-sync-get-dataset-items💼 Target Roles: DevOps, AWS Cloud Engineer, Platform Engineer.⚡ Batch Sizing: Configured with "maxResultsPerQuery": 5 during testing to manage LLM rate limits.2️⃣ JavaScript Pre-Processing & Data NormalizationA custom Code Node (JavaScript) sanitizes the raw HTML/JSON output from Naukri, handles missing fields with safe fallbacks, cleans up text formatting, and slices the payload for execution safety.// n8n Code Node - JavaScript Pre-processing
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
3️⃣ AI Match Evaluation (Google Gemini Agent)The AI Agent node combines candidate specifications with job descriptions and passes them to the connected Google Gemini Chat Model sub-node (models/gemini-1.5-flash or models/gemini-3.5-flash).👤 Candidate Profile Prompt:Role: Technical Recruiter EvaluatorCandidate Profile: 1+ years experience in AWS Cloud & DevOps Engineering. Core skills include AWS (EC2, S3, VPC, IAM, RDS, Lambda, CloudWatch), Terraform, Docker, Kubernetes, CI/CD Pipelines (Jenkins / GitHub Actions), Python, Shell Scripting, and Linux Administration.Evaluation Criteria: Analyze candidate alignment against the scraped jdText. Output strict JSON specifying matchScore (0–100), seniorityFit, matchingSkills, missingSkills, and concise reasoning.4️⃣ Output Parsing & Match FilteringThe raw text returned by the AI is parsed back into JSON by Code in JavaScript1. The code applies a strict fit filter (e.g., matchScore >= 50) to filter out irrelevant or over-senior roles.// n8n Code Node - JavaScript Post-processing
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
5️⃣ Data Persistence & Alerting📊 Google Sheets Tracking: All qualified matches are appended to a central Google Sheet job tracking table.📬 Gmail Digest Notification: An instant email digest is dispatched to the user containing score breakdowns and key skill fits.🛠️ Troubleshooting & Technical RetrospectiveIssue EncounteredRoot CauseResolution Implemented404 Actor Not FoundIncorrect Apify owner slug (apify~naukri-scraper)Switched API endpoint to active author namespace (epicscrapers~naukri-scraper).429 Quota ExceededPassing 40 items simultaneously into Gemini Free TierAdded .slice(0, 5) in JS pre-processing and configured 15s retry waits.404 Model Not FoundDeprecated model endpoint (gemini-2.5-flash)Updated Gemini sub-node to stable gemini-1.5-flash / gemini-3.5-flash.503 Service UnavailableGoogle Cloud API capacity spikesEnabled Retry On Fail (Max Tries: 5, Wait: 5000ms) in Node Settings.Forbidden Gmail APIUnenabled API / missing OAuth scopesEnabled Gmail API in Google Cloud Console project and re-authorized n8n credentials.🚀 Getting Started & Setup📥 Clone the Repository:git clone https://github.com/your-username/n8n-naukri-devops-job-agent.git
cd n8n-naukri-devops-job-agent
⚙️ Import Workflow:Open your local or cloud n8n instance.Select Workflows ➔ Import from File.Choose workflows/naukri-devops-agent.json.🔑 Configure Credentials:Add your Apify API Token in HTTP Request1.Add your Google Gemini API Key in Google Gemini Chat Model.Authorize Google Sheets and Gmail OAuth2 connections.🟢 Activate:Toggle the Publish switch to enable scheduled execution.
