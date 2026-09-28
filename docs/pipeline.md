# Pipeline Documentation

## Overview

Data Career Copilot is implemented as an n8n workflow that transforms raw job-search results into prioritized opportunities for human review.

The production pipeline follows eight functional stages:

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
08. Email Summary
```

The workflow deliberately separates deterministic processing from semantic AI tasks. Rules handle operations that should be repeatable and auditable; LLMs are used where natural-language interpretation adds value.

## Execution Model

The workflow is scheduled in n8n and runs twice per week: Tuesday and Friday at 10:00 in `America/Mexico_City`, using a local n8n runtime containerized with Docker.

Each execution discovers vacancies, removes irrelevant or previously processed records, evaluates new opportunities one at a time, stores the results, and generates a final prioritized email summary.

---

## 01. Job Discovery

### Schedule Trigger
Starts the workflow according to the configured schedule.

### Generar Consultas de Búsqueda
Creates five search families: Data Analyst, Business Intelligence, Operations Analytics, Business Analytics, and Data + Automation. Multiple search families improve coverage because relevant vacancies do not use a single standardized title.

### Consultar Vacantes en Google Jobs
Sends each query to SerpApi using the Google Jobs engine with Mexico City / Mexico localization. The public workflow references `$env.SERPAPI_API_KEY` rather than embedding the credential.

### Separar Vacantes Encontradas
Converts SerpApi result collections into individual n8n items, performs an initial in-memory duplicate check, and preserves the originating query as `query_origen`.

**Output:** individual raw job records ready for normalization.

---

## 02. Normalization & Relevance Filtering

### Normalizar Vacante Google Jobs
Converts provider-specific fields into a consistent internal vacancy structure, isolating downstream logic from the original SerpApi response format.

### Generar Hash de Vacante
Creates a deterministic `job_hash` as an additional identity signal alongside `source_job_id`.

### Filtrar Vacantes Relevantes
The **Data Relevance Filter V1.1** is a deterministic JavaScript pre-filter. It evaluates title and description signals before any LLM call.

It considers target Data/BI roles, ambiguous analytical titles, Data and BI tools, analysis responsibilities, reporting and KPIs, data preparation, business/operations context, and automation. It also blocks clearly non-priority technical profiles and detects vacancies whose central function is strongly Machine Learning or Data Science oriented.

The principal match paths are:

```text
Direct target role
Ambiguous title + sufficient Data evidence
Data role inferred from strong analytical content
Hybrid Data + Automation role
```

### Validar Relevancia de Vacante
Routes only relevant vacancies forward. Irrelevant results stop before expensive AI processing.

**Design rationale:** deterministic filtering is cheaper, faster, and more reproducible than sending every raw result to an LLM.

---

## 03. Deduplication

### Leer Vacantes Existentes del CRM
Reads previously processed vacancies from the Google Sheets CRM.

### Deduplicar Vacantes contra CRM
Compares current vacancies against existing `source_job_id` and `job_hash` values, as well as identifiers already seen during the current execution.

Only genuinely new vacancies continue.

**Why this matters:** duplicate vacancies should not generate repeated LLM calls, CRM records, or recommendations.

---

## 04. AI Job Extraction

### Procesar Vacantes Relevantes
A Loop Over Items node processes new relevant vacancies individually with a batch size of one.

### Preparar Vacante SerpApi para Análisis
Explicitly selects the normalized fields provided to the extraction model. Prefilter diagnostics are intentionally excluded so the relevance decision does not bias extraction.

### Analizar Vacante con IA
Transforms the unstructured vacancy into a standardized schema. The model performs **extraction and classification only**, not candidate evaluation.

The schema includes position, company, location, modality, salary, currency, seniority, required experience, English, languages, databases, BI tools, other tools, Data competencies, responsibilities, mandatory/desirable/exclusionary requirements, core technologies, and a technical summary.

### Structured Output Parser
Constrains the LLM response to a predictable machine-readable schema required by downstream nodes.

---

## 05. Candidate Profile

### Leer Perfil Candidato
Reads the candidate profile from a dedicated Google Sheets table rather than hard-coding it into workflow logic.

### Estructurar Perfil Candidato
Transforms the source `campo | valor` rows into a single JavaScript object.

```text
Google Sheets rows
        ↓
campo | valor
        ↓
Structured candidate object
```

No matching logic is performed here.

**Design rationale:** candidate information can evolve independently from workflow code.

---

## 06. AI Matching + Deterministic Scoring

### Analizar Compatibilidad con Perfil
Compares the structured candidate profile with the structured vacancy.

The LLM evaluates relevant experience, tools and technologies, responsibilities, industry/context, seniority, language, location/modality, and salary. It also identifies strengths, missing skills, critical missing requirements, missing core technologies, and explicit exclusionary requirements.

The prompt distinguishes total professional experience from direct Data experience and recognizes project work as practical evidence without treating it as equivalent to employment experience.

### Structured Compatibility Output
Constrains the matching response. The LLM does **not** assign the final numerical score or final category.

### Calcular Score de Compatibilidad
JavaScript converts the qualitative evaluation into a deterministic score.

| Dimension | Weight |
|---|---:|
| Relevant experience | 25 |
| Tools and technologies | 25 |
| Responsibilities | 15 |
| Industry / context | 10 |
| Seniority | 10 |
| Language | 5 |
| Location / modality | 5 |
| Salary | 5 |

Explicit penalties and score caps handle critical requirements, missing core technologies, exclusionary requirements, and insufficient tool compatibility.

```text
80–100  → Apply
60–79   → Review
0–59    → Discard
```

The base score remains separate from the final score so penalties and caps remain inspectable.

### Integrar Scoring en Vacante
Merges vacancy data, structured extraction, compatibility analysis, and deterministic scoring into the record that will be persisted.

---

## 07. CRM Persistence

### Generar ID de Vacante
Creates a workflow-level vacancy identifier before persistence.

### Guardar Vacante en CRM
Appends the final structured record to Google Sheets, including vacancy data, technical requirements, compatibility analysis, score, recommendation, source metadata, application URL, and execution metadata.

Application date, notes, and comments remain intentionally manual. After persistence, the workflow returns to the loop.

---

## 08. Execution Summary & Notification

### Preparar ID de Ejecución
Prepares the execution identifier used to retrieve records generated during the current run.

### Buscar Vacantes de esta Ejecución
Queries the CRM for vacancies associated with the current execution so historical opportunities are not mixed with new results.

### Preparar Resumen de Vacantes
Sorts vacancies by compatibility and groups them into `Apply`, `Review`, and `Discard`.

Detailed cards are generated for **Apply** and **Review**. Discarded vacancies are counted for execution-level visibility but omitted from the detailed list.

### Gmail Notification
Sends the final HTML summary. The subject reports the number of opportunities available to apply to and review.

---

## Data Flow Summary

```text
Schedule
   ↓
Generate Search Families
   ↓
Google Jobs via SerpApi
   ↓
Split Results
   ↓
Normalize
   ↓
Generate Job Hash
   ↓
Deterministic Relevance Filter
   ↓
Read Existing CRM
   ↓
Deduplicate
   ↓
Loop New Vacancies
   ↓
Structured Job Extraction
   ↓
Read + Structure Candidate Profile
   ↓
Semantic Compatibility Analysis
   ↓
Deterministic Score
   ↓
Persist to CRM
   ↓
Repeat Until Complete
   ↓
Retrieve Current Execution
   ↓
Prepare Prioritized Summary
   ↓
Email Notification
```

## Processing Responsibilities

| Responsibility | Implementation |
|---|---|
| Scheduling | n8n |
| Search-family generation | JavaScript |
| Job ingestion | SerpApi / Google Jobs |
| Normalization | n8n + JavaScript |
| Relevance filtering | Deterministic JavaScript |
| Deduplication | JavaScript + Google Sheets CRM |
| Vacancy interpretation | OpenAI + Structured Output |
| Candidate data | Google Sheets |
| Semantic compatibility | OpenAI + Structured Output |
| Numerical scoring | Deterministic JavaScript |
| Persistence | Google Sheets |
| Notification | Gmail |

The division is intentional: AI is used for semantic interpretation, while workflow control, filtering, deduplication, and scoring remain explicit and reproducible.

## Failure Boundaries and Known Limitations

Known limitations include incomplete or ambiguous source descriptions, inconsistent salary or modality formatting, semantic variation between LLM executions, possible relevance-filter false positives/negatives, and dependency on the current job-discovery provider.

The architecture reduces their impact through structured outputs, deterministic scoring, source traceability, and human review as the final decision layer.

## Related Documentation

- [Project README](../README.md)
- [System Architecture](architecture.md)
- [Sanitized n8n Workflow](../workflows/data-career-copilot.sanitized.json)
