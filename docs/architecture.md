\# System Architecture



\## Overview



Data Career Copilot is an automated job intelligence and decision-support system designed to reduce the manual effort required to discover and evaluate Data \& Analytics job opportunities.



The system combines deterministic rules, LLM-based semantic analysis, structured candidate data, and rule-based scoring in a single automated pipeline orchestrated with n8n.



A central design principle is that the LLM does not make the final scoring decision. AI is used where semantic interpretation is valuable, while deterministic logic is used where consistency, traceability, and reproducibility are more important.



\---



\## High-Level Architecture



```text

┌─────────────────────────────┐

│  01. Job Discovery          │

│  SerpApi / Google Jobs      │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  02. Normalization \&        │

│      Relevance Filtering    │

│  JavaScript Rules           │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  03. Deduplication          │

│  CRM + Job Hash             │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  04. AI Job Extraction      │

│  OpenAI + Structured Output │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  05. Candidate Profile      │

│  Google Sheets              │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  06. AI Matching            │

│  + Deterministic Scoring    │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  07. CRM Persistence        │

│  Google Sheets              │

└──────────────┬──────────────┘

&#x20;              │

&#x20;              ▼

┌─────────────────────────────┐

│  08. Email Summary          │

│  Gmail                      │

└─────────────────────────────┘

Pipeline Components

01\. Job Discovery

The workflow starts automatically on a predefined schedule.

A set of search queries targets several families of Data \& Analytics roles, including:

\- Data Analyst

\- Business Intelligence Analyst

\- Operations Analytics

\- Business Analytics

\- Data \& Automation roles

The queries are sent to SerpApi using the Google Jobs engine.

This stage acts as the ingestion layer of the system.

Output

Raw job postings containing information such as:

\- job title

\- company

\- location

\- description

\- publication date

\- employment type

\- application source

\- application URL

\- originating search query

02\. Normalization \& Relevance Filtering

Raw search results are transformed into a consistent internal structure before any LLM processing occurs.

A deterministic JavaScript relevance filter evaluates each vacancy using signals from the job title and description.

The filter is designed to identify roles related to:

\- Data Analytics

\- Business Intelligence

\- Operations Analytics

\- Business Analytics

\- hybrid Data + Automation work

It also detects signals associated with positions outside the primary scope of the project, such as strongly specialized Machine Learning or Data Science roles.

Why filter before the LLM?

Sending every search result directly to an LLM would:

\- increase API consumption

\- process irrelevant positions

\- introduce unnecessary semantic variability

\- increase execution time

The deterministic pre-filter therefore acts as a low-cost first decision layer.

03\. Deduplication

Before expensive AI analysis occurs, vacancies are compared against the existing CRM.

The workflow uses source identifiers and a generated job hash to prevent the same opportunity from being processed repeatedly.

This stage reduces:

\- duplicate CRM records

\- repeated LLM calls

\- unnecessary API consumption

\- duplicated opportunities in email summaries

04\. AI Job Extraction

Vacancy descriptions are written in unstructured natural language and vary significantly between employers.

An LLM transforms this information into a structured job schema.

The extraction layer identifies fields such as:

\- position

\- company

\- location

\- work modality

\- salary

\- seniority

\- required experience

\- English requirements

\- programming languages

\- databases

\- BI tools

\- other tools

\- data competencies

\- responsibilities

\- mandatory requirements

\- desirable requirements

\- exclusionary requirements

\- core technologies

Structured Outputs are used to enforce a predictable response schema.

Role of AI in this stage

The LLM performs semantic extraction, not candidate evaluation.

It interprets the vacancy and converts unstructured job descriptions into structured information that downstream components can process consistently.

05\. Candidate Profile

Candidate information is stored independently from the workflow logic.

The profile includes information such as:

\- target roles

\- professional experience

\- analytical experience

\- technical skills

\- skill proficiency levels

\- English level

\- salary expectations

\- preferred work modalities

\- geographic restrictions

\- professional constraints

The profile is stored in Google Sheets and transformed into a structured object during execution.

Design rationale

Separating candidate information from workflow code allows the profile to evolve without modifying the automation itself.

This improves maintainability and creates a clearer separation between:

Candidate Data

&#x20;     ↓

Evaluation Logic

&#x20;     ↓

Job Opportunities



06\. AI Matching + Deterministic Scoring

This is the main decision-support layer of the system.

The structured vacancy is compared with the structured candidate profile.

The LLM evaluates semantic compatibility across several dimensions:

\- relevant experience

\- tools and technologies

\- responsibilities

\- industry context

\- seniority

\- language

\- location and modality

\- salary

The LLM also identifies:

\- strengths

\- missing skills

\- critical missing requirements

\- missing core technologies

\- explicit exclusionary requirements

However, the LLM does not generate the final compatibility score.

Deterministic Scoring

After semantic evaluation, JavaScript calculates the compatibility score using predefined weights.

Dimension	Weight

Relevant experience	25

Tools and technologies	25

Responsibilities	15

Industry context	10

Seniority	10

Language	5

Location / modality	5

Salary	5





Additional penalties and score caps are applied when critical conditions are detected.

Examples include:

\- missing core technologies

\- critical mandatory requirements

\- exclusionary requirements

\- insufficient tool compatibility

The final result is classified into:

Apply

Review

Discard



Why use hybrid scoring?

The architecture deliberately separates two types of reasoning.

LLM

Good at interpreting:

\- ambiguous job descriptions

\- semantic similarity

\- transferable experience

\- natural-language requirements

Deterministic code

Better suited for:

\- numerical scoring

\- thresholds

\- penalties

\- caps

\- reproducibility

This prevents the LLM from freely inventing a compatibility percentage and makes the final score easier to audit.

07\. CRM Persistence

Processed vacancies are stored in a Google Sheets CRM.

The CRM contains:

\- normalized vacancy information

\- extracted technical requirements

\- compatibility analysis

\- compatibility score

\- recommendation

\- application URL

\- processing metadata

Some fields are intentionally reserved for manual candidate actions, such as:

\- application date

\- notes

\- comments

This allows automated intelligence and human decision-making to coexist in the same workflow.

08\. Email Summary

After all vacancies from an execution have been processed, the workflow retrieves the jobs associated with that execution.

Vacancies are sorted by compatibility score.

The email prioritizes:

Apply

Review



while discarded opportunities are summarized only as a count.

Each surfaced opportunity includes relevant decision information such as:

\- position

\- company

\- location

\- modality

\- salary

\- compatibility score

\- recommendation

\- strengths

\- critical gaps

\- missing core technologies

\- application URL

The objective is not to replace the candidate's decision, but to reduce the number of vacancies that require manual review.

Architectural Pattern

The system can be summarized as:

DISCOVER

&#x20;  ↓

FILTER

&#x20;  ↓

DEDUPLICATE

&#x20;  ↓

STRUCTURE

&#x20;  ↓

COMPARE

&#x20;  ↓

SCORE

&#x20;  ↓

STORE

&#x20;  ↓

NOTIFY



Three different processing approaches are intentionally combined:

Deterministic Rules

&#x20;       +

LLM Semantic Reasoning

&#x20;       +

Deterministic Scoring



This hybrid architecture allows the system to use AI where language interpretation adds value while retaining explicit business rules for decisions that should remain reproducible.

Deployment Architecture

The current version runs n8n locally using Docker.

Windows Host

&#x20;    │

&#x20;    ▼

Docker

&#x20;    │

&#x20;    ▼

n8n Container

&#x20;    │

&#x20;    ├── SerpApi

&#x20;    ├── OpenAI API

&#x20;    ├── Google Sheets

&#x20;    └── Gmail



The n8n runtime uses persistent Docker storage so workflow configuration survives container recreation.

Secrets such as the SerpApi API key are provided through environment variables rather than being embedded directly in the public workflow.

Example:

SERPAPI\_API\_KEY=your\_serpapi\_api\_key\_here



The public workflow therefore references:

$env.SERPAPI\_API\_KEY



instead of containing the real credential.

Current Deployment Decision

The workflow currently runs locally because its workload does not require continuous cloud infrastructure.

The scheduled process only needs to execute a limited number of times per week, making local containerized execution a cost-efficient deployment strategy.

Docker still provides:

\- environment isolation

\- persistent workflow data

\- reproducibility

\- portability

\- straightforward migration

If higher availability becomes necessary, the same architecture can later be migrated to a VPS or managed n8n environment without redesigning the core pipeline.

Design Principles

The architecture follows several principles:

1\. Filter before expensive processing.

2\. Deduplicate before calling the LLM.

3\. Use AI for semantic interpretation, not arbitrary scoring.

4\. Keep scoring rules explicit and auditable.

5\. Separate candidate data from workflow logic.

6\. Keep human decision-making in the final application process.

7\. Store structured outputs for later analysis.

8\. Keep credentials outside public workflow files.

These principles allow Data Career Copilot to function not only as an automation workflow, but as a reproducible job intelligence pipeline.


