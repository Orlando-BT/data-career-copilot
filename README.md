# Data Career Copilot

> AI-powered job intelligence and career decision-support system for Data & Analytics roles.

**Current V1:** automated job discovery, relevance filtering, matching, prioritization, CRM persistence, and opportunity notifications.

**Long-term vision:** evolve into an AI Career Copilot that supports candidates throughout the journey from opportunity discovery to personalized preparation and interview readiness.

---

## Problem / Need

Searching for a job can become a repetitive and time-consuming process.

Finding relevant opportunities requires searching multiple role variations, reviewing job descriptions, identifying technical requirements, discarding irrelevant positions, checking previously reviewed vacancies, and comparing each role against the candidate's actual skills and constraints.

In my case, that time competed directly with another priority: continuing to develop the technical skills required for Data & Analytics roles, including **Python, SQL, Power BI, and automation**.

Data Career Copilot was created to automate the repetitive parts of the job-search process so that more time can be invested in **learning, portfolio projects, and reviewing the opportunities that are actually worth considering**.

The system supports the decision process; it does not make career decisions on behalf of the candidate.

---

## Project Question

**How can I automate the discovery and initial evaluation of Data & Analytics job opportunities so that I can focus my time on developing technical skills and reviewing the opportunities that are actually worth considering?**

### Answer

I built an end-to-end job intelligence pipeline that automatically discovers vacancies, removes irrelevant and duplicate results, converts unstructured job descriptions into structured data, evaluates compatibility against a configurable candidate profile, calculates a deterministic compatibility score, stores the results in a lightweight CRM, and sends a prioritized email summary.

---

## Solution Overview

The pipeline is organized into eight stages:

```text

01. Job Discovery

        ↓

02. Normalization & Relevance Filtering

        ↓

03. Deduplication

        ↓

04. AI Job Extraction

        ↓

05. Candidate Profile

        ↓

06. AI Matching + Deterministic Scoring

        ↓

07. CRM Persistence

        ↓

08. Prioritized Email Summary

```

The architecture combines **deterministic business rules** with **LLM-based semantic analysis**.

Instead of asking an LLM to control the entire process, each technology is used where it provides the most value:

```text

Job Sources

     ↓

Normalization

     ↓

Deterministic Relevance Filter

     ↓

Deduplication

     ↓

LLM Structured Extraction

     ↓

Candidate Profile

     ↓

LLM Semantic Matching

     ↓

Deterministic Scoring

     ↓

CRM

     ↓

Prioritized Notification

```

---

## Technology Stack

| Technology | Purpose |

|---|---|

| **n8n** | Workflow orchestration and scheduling |

| **SerpApi / Google Jobs** | Job discovery |

| **JavaScript** | Normalization, relevance filtering, deduplication, and scoring |

| **OpenAI** | Structured job extraction and semantic compatibility analysis |

| **Structured Outputs** | Consistent, machine-readable LLM responses |

| **Google Sheets** | Candidate profile and lightweight job-search CRM |

| **Gmail** | Prioritized opportunity summaries |

| **Docker** | Portable local n8n runtime |

| **Git / GitHub** | Version control and project documentation |

---

# How It Works

## 1. Job Discovery

The workflow generates multiple search families targeting Data & Analytics roles and retrieves vacancies from Google Jobs through SerpApi.

The search strategy covers role families such as:

- Data Analyst

- Business Intelligence Analyst

- Operations Analytics

- Business Analytics

- Data & Automation roles

Using multiple search families increases coverage without relying on a single job-title formulation.

---

## 2. Normalization & Relevance Filtering

Before any LLM analysis, vacancies pass through deterministic JavaScript logic and the **Data Relevance Filter V1.1**.

The filter evaluates target role families, analytical signals, hybrid Data + Automation patterns, and exclusion conditions.

Clearly irrelevant vacancies are removed before reaching the AI stage.

```text

Raw Jobs

   ↓

Normalization

   ↓

Relevance Rules

   ↓

Relevant? ── No ──→ Discard

   │

  Yes

   ↓

Continue Pipeline

```

This design reduces noise and avoids spending LLM calls on vacancies that are already outside the target scope.

---

## 3. Deduplication

Vacancies are normalized and assigned identifiers that can be compared against previously processed jobs stored in the CRM.

```text

New Vacancy

     ↓

Generate Identifier

     ↓

Compare with CRM

     ↓

Already Processed?

   ↙             ↘

 Yes              No

  ↓                ↓

Stop          AI Analysis

```

Deduplication happens **before expensive AI processing**, preventing repeated analysis and unnecessary API usage.

---

## 4. Structured AI Job Extraction

Relevant job descriptions are analyzed using an LLM.

Instead of requesting a free-form summary, the workflow enforces a structured output schema.

The system extracts information such as:

- programming and query languages

- databases

- BI tools

- data competencies

- responsibilities

- mandatory requirements

- desirable requirements

- exclusionary requirements

- core technologies

- seniority

- years of experience

- English requirements

- work modality

- salary information

This converts unstructured job descriptions into standardized information that downstream nodes can process programmatically.

---

## 5. Candidate Profile

The candidate profile is stored separately from the workflow logic and retrieved dynamically from Google Sheets.

The profile contains information such as:

```text

Professional Experience

Technical Skills

Tools

Data & Analytics Experience

English Level

Target Roles

Salary Expectations

Location

Work Modality

Professional Constraints

```

Separating the candidate profile from the workflow makes the system easier to maintain.

Skills, experience, preferences, and constraints can evolve without rewriting the core pipeline.

---

## 6. Hybrid AI Matching & Deterministic Scoring

One of the main architectural decisions in the project is separating **semantic interpretation** from **numerical scoring**.

### LLM responsibility

The LLM compares the structured vacancy against the candidate profile and evaluates dimensions such as:

- relevant experience

- tools and technologies

- responsibilities

- industry context

- seniority

- language

- location and modality

- salary

It also identifies:

- strengths

- missing skills

- critical missing requirements

- missing core technologies

- explicit exclusionary conditions

### Deterministic scoring responsibility

The LLM does **not** generate the final compatibility percentage.

A JavaScript scoring layer converts the structured evaluation into a deterministic score.

The current scoring model uses:

| Dimension | Weight |

|---|---:|

| Relevant experience | 25 |

| Tools & technologies | 25 |

| Responsibilities | 15 |

| Industry / context | 10 |

| Seniority | 10 |

| Language | 5 |

| Location / modality | 5 |

| Salary | 5 |

Additional rules apply penalties and score caps when critical requirements or core technologies are missing.

The final result is classified into three actionable categories:

```text

80–100  → APPLY

60–79   → REVIEW

0–59    → DISCARD

```

This hybrid approach makes the final prioritization more **reproducible, explainable, and auditable** than asking an LLM to invent a compatibility percentage directly.

---

## 7. CRM Persistence

Processed vacancies and their evaluation results are stored in Google Sheets.

The spreadsheet works as a lightweight job-search CRM containing information such as:

```text

Vacancy

Company

Location

Modality

Salary

Requirements

Technologies

Compatibility Score

Recommendation

Strengths

Skill Gaps

Application URL

Application Status

Notes

```

This persistence layer also allows future executions to identify previously processed opportunities.

Manual fields remain available for application tracking and personal notes.

---

## 8. Prioritized Email Summary

At the end of each execution, the workflow generates an HTML email containing the most relevant opportunities.

Vacancies classified as:

```text

APPLY

REVIEW

```

are included as detailed opportunity cards.

Jobs classified as:

```text

DISCARD

```

are counted for execution-level visibility but excluded from the detailed recommendation list.

Each opportunity can include:

```text

Job Title

Company

Location

Work Modality

Salary

Compatibility Score

Recommendation

Strengths

Critical Gaps

Missing Core Technologies

Reason for Recommendation

Application URL

```

The goal is not to replace human judgment, but to significantly reduce the amount of information that needs to be reviewed manually.

---

# Validation

The workflow has been tested using real job-search results.

## End-to-End Execution

One full execution processed:

| Result | Vacancies |

|---|---:|

| New vacancies processed | **47** |

| Apply | **9** |

| Review | **7** |

| Discard | **31** |

| Opportunities surfaced for review | **16** |

Instead of manually reviewing all 47 new vacancies with the same level of attention, the pipeline surfaced 16 opportunities classified as Apply or Review for prioritized human evaluation.

> These numbers represent one validated execution and should not be interpreted as a global accuracy measurement of the system.

---

## Relevance Filter Validation

The relevance-filtering component was also evaluated separately using a labeled sample of job vacancies.

A previous baseline evaluation produced approximately:

| Metric | Result |

|---|---:|

| Precision | **91.7%** |

| Recall | **97.8%** |

| F1 Score | **94.6%** |

These results were used diagnostically to identify false positives and false negatives and informed the evolution toward **Data Relevance Filter V1.1**.

They should not be interpreted as universal production performance metrics because the evaluation was performed on a limited labeled sample.

The current development approach favors **evidence-based iteration**:

> isolated anomalies are documented and observed, while filtering or scoring rules are modified only when an error pattern repeats across independent executions.

This reduces the risk of overfitting the workflow to individual vacancies.

---

# Key Design Decisions

## Why not use AI for everything?

Not every task requires an LLM.

Deterministic code handles:

```text

Normalization

Filtering

Deduplication

Scoring

Workflow Control

```

while the LLM is reserved for tasks that benefit from semantic interpretation:

```text

Understanding Job Descriptions

Extracting Requirements

Comparing Candidate and Vacancy Context

Identifying Strengths and Gaps

```

This creates a hybrid architecture where AI complements traditional programming instead of replacing it.

---

## Why calculate the score outside the LLM?

A freely generated compatibility percentage would be difficult to reproduce and audit.

The LLM therefore produces structured qualitative assessments while explicit JavaScript rules calculate the final score.

```text

LLM

 ↓

Structured Evaluation

 ↓

Deterministic Rules

 ↓

Compatibility Score

```

The result is easier to inspect, test, and modify.

---

## Why filter vacancies before AI analysis?

Clearly irrelevant vacancies do not require semantic analysis.

```text

100 Raw Jobs

      ↓

Deterministic Filtering

      ↓

Relevant Jobs Only

      ↓

LLM Processing

```

This reduces unnecessary API usage and keeps AI focused on the jobs where semantic interpretation provides value.

---

## Why deduplicate before AI processing?

Analyzing the same vacancy multiple times would increase API usage without adding new information.

Deduplication therefore happens before the LLM processing loop.

---

## Why separate the candidate profile?

Candidate information changes independently from workflow logic.

Keeping the profile outside the pipeline means that skills, experience, salary expectations, location constraints, or target roles can be updated without modifying the workflow architecture.

---

## Why run n8n locally with Docker?

The current workload does not require always-on cloud infrastructure.

Running n8n locally with Docker keeps infrastructure costs low while providing:

- environment isolation

- persistent workflow data

- portability

- reproducible deployment

- control over the runtime environment

For the current execution frequency, this provides a favorable **cost-benefit ratio**.

The containerized architecture also preserves a clear migration path:

```text

Local Docker

     ↓

VPS

     ↓

Managed n8n / Cloud Infrastructure

```

If higher availability or execution frequency becomes necessary, the workflow can migrate to an always-on environment without redesigning the business logic.

---

# Conclusions

The project showed that job matching benefits from a **hybrid architecture** rather than delegating the entire process to an LLM.

Deterministic logic is well suited to repeatable operations such as filtering, deduplication, and scoring, while an LLM adds value when interpreting unstructured job descriptions and comparing semantic requirements.

The validation process also showed that **technical similarity alone is not sufficient to identify a suitable opportunity**.

A vacancy can have strong technical alignment while still being unsuitable because of factors such as:

- mandatory technologies

- required experience level

- language requirements

- academic requirements

- work modality

- geographic constraints

For this reason, Data Career Copilot is designed as a **decision-support system**, not an autonomous career decision-maker.

The system automates information processing and prioritization while leaving the final application decision to the candidate.

---

# Current Limitations

The current version has several known limitations:

- Job descriptions can be incomplete or ambiguous.

- Source data can contain inconsistent salary or modality information.

- Semantic evaluations can vary between LLM executions.

- Deterministic relevance filtering can produce false positives or false negatives.

- The current validation sample is not sufficient to claim overall matching accuracy.

- V1 currently depends on a single job-discovery provider.

Local Docker execution is treated as a **cost-conscious deployment decision rather than a functional limitation**.

The trade-off is host availability, while containerization preserves portability to always-on infrastructure if that requirement emerges.

---

# Future Roadmap

The current version focuses on:

```text

DISCOVER → EVALUATE → PRIORITIZE

```

The long-term vision is broader.

Data Career Copilot could evolve into an end-to-end career assistant that supports candidates throughout the journey from discovering an opportunity to preparing for the hiring process.

```text

DISCOVER

    ↓

EVALUATE

    ↓

ADAPT

    ↓

PREPARE

    ↓

PRACTICE

    ↓

APPLY

    ↓

LEARN

```

Potential future capabilities include:

### Multi-Source Job Ingestion

Integrate additional job boards, APIs, and permitted job-data sources into a common normalization pipeline.

```text

Job Source A ─┐

Job Source B ─┼──→ Normalization → Deduplication → Analysis

Job Source C ─┘

```

This would reduce dependency on a single discovery provider and increase vacancy coverage.

### Job-Specific CV Generation

For selected opportunities, the system could generate a tailored version of the candidate's CV using:

```text

Candidate Profile

+

Original CV

+

Vacancy Requirements

+

Strengths

+

Relevant Experience

```

The LLM would be allowed to restructure, prioritize, and improve presentation, but **never fabricate experience, skills, education, or achievements**.

### Application Tracking

Extend the current CRM to monitor:

```text

Opportunity

→ Applied

→ Recruiter Contact

→ Interview

→ Technical Interview

→ Offer / Rejected

```

This would make it possible to analyze the complete job-search funnel.

### Personalized Interview Preparation

The requirements extracted from each vacancy could be used to automatically identify what the candidate should practice before an interview.

For example:

```text

Vacancy Requirements

        +

Candidate Profile

        ↓

Skill Gap Analysis

        ↓

Personalized Practice

```

### Multiple Interview Modes

Practice sessions could be generated for different stages of a hiring process:

- Technical interview

- People / HR interview

- Hiring manager interview

- Behavioral interview

- Role-specific scenarios

The practice content would be generated using the actual vacancy requirements rather than generic interview questions.

### Actionable Opportunity Notifications

Future email notifications could include actions such as:

```text

\[ View Vacancy ]

\[ Generate Tailored CV ]

\[ Practice Technical Interview ]

\[ Practice HR Interview ]

```

Each action could trigger a specialized workflow.

This would transform the notification layer from a passive report into an entry point for additional career workflows.

### Interview Feedback & Skill-Gap Analysis

Practice sessions could generate structured feedback identifying:

```text

Strong Areas

Weak Areas

Missing Concepts

Communication Gaps

Technical Topics to Review

```

Those results could then generate another preparation cycle.

### Career Feedback Loop

Application and interview outcomes could eventually become new input data.

```text

Job Discovery

      ↓

Matching

      ↓

Application

      ↓

Interview

      ↓

Outcome

      ↓

Feedback

      ↓

Better Career Intelligence

```

This would allow future versions to analyze which skills, roles, companies, and opportunity characteristics lead to better outcomes.

### Labor-Market Analytics

Historical vacancy data could also support analytics such as:

- most requested technologies

- recurring skill gaps

- salary distributions

- demand by role family

- remote vs. hybrid opportunities

- required experience trends

- candidate compatibility trends

- application conversion rates

A future Power BI or analytics layer could turn the accumulated CRM data into labor-market intelligence.

### Cloud Deployment

If higher availability or execution frequency becomes necessary, the current containerized architecture can migrate from local Docker to:

- a VPS

- managed Docker infrastructure

- managed n8n hosting

without changing the fundamental workflow design.

---

# Security

Secrets are not stored directly in the public workflow.

For example, the SerpApi credential is retrieved through the environment variable:

```text

SERPAPI_API_KEY

```

The repository includes an `.env.example` file documenting the required environment variable without exposing the real credential.

```env

SERPAPI_API_KEY=your_serpapi_api_key_here

```

Public workflow exports must be sanitized before publication to remove:

- API keys

- personal email addresses

- Google Sheets document IDs

- credential IDs

- credential names

- instance-specific identifiers

- environment-specific metadata

The production workflow should never be published directly without this sanitization step.

---

# Deployment

V1 runs in a local Docker environment with n8n and is scheduled for automated execution.

The deployment strategy was selected to keep infrastructure costs proportional to the current workload while retaining portability.

```text

Windows Host

     ↓

Docker

     ↓

n8n

     ↓

Data Career Copilot

```

The environment uses persistent Docker storage so workflow configuration survives container recreation.

Detailed setup and deployment instructions are planned for a future `docs/local-deployment.md` document.

---

# Repository Structure

```text

data-career-copilot/

│

├── README.md

├── .gitignore

├── .env.example

│

├── workflows/

│   └── data-career-copilot.sanitized.json

│

├── docs/

│   ├── architecture.md

│   ├── pipeline.md

│   ├── relevance-filter.md

│   ├── matching-scoring.md

│   ├── validation.md

│   ├── local-deployment.md

│   └── security.md

│

└── assets/

    └── architecture-diagram.png

```

The public workflow will contain the system logic while excluding private credentials and environment-specific identifiers.

---

# Project Status

### V1 — Functional

The end-to-end pipeline is operational and scheduled for automated execution.

Current V1 covers:

```text

✓ Job Discovery

✓ Normalization

✓ Relevance Filtering

✓ Deduplication

✓ Structured AI Extraction

✓ Candidate Profile Retrieval

✓ Semantic Matching

✓ Deterministic Scoring

✓ CRM Persistence

✓ Prioritized Email Notification

✓ Scheduled Execution

✓ Local Docker Deployment

```

Future iterations will focus on **evaluation, observability, additional ingestion sources, application tracking, CV personalization, and interview preparation** rather than expanding the workflow without measurable evidence.

---

## Author

**Orlando Bautista Trejo**

Project developed as part of a portfolio focused on **Data Analytics, AI Automation, and intelligent operational systems**.
