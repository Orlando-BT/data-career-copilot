# Relevance Filter

## 1. Purpose

Data Career Copilot retrieves job postings from Google Jobs through SerpApi. However, not every result returned by a search query is sufficiently relevant to justify further processing with an LLM.

The Relevance Filter acts as an early deterministic classification layer that decides whether a job posting should continue through the pipeline.

Its main objective is to:

> **Reduce irrelevant vacancies before expensive AI processing while preserving as many potentially valuable Data & Analytics opportunities as possible.**

The filter is intentionally executed before LLM-based job extraction and candidate matching.

This creates a two-stage decision architecture:

```text
Job Discovery
      ↓
Deterministic Relevance Filter
      ↓
Relevant Vacancies
      ↓
LLM Job Extraction
      ↓
Candidate Matching
```

The filter does **not** determine whether the candidate should apply.

It only answers an earlier question:

> Is this vacancy sufficiently related to the target Data & Analytics job families to justify deeper analysis?

---

## 2. Why Filter Before the LLM?

Sending every retrieved vacancy directly to an LLM would be technically possible, but inefficient.

Search engines can return jobs that contain relevant keywords without actually representing a target role. Examples include vacancies centered on:

- Data Science
- Machine Learning
- Data Engineering
- Software Development
- highly specialized technical functions
- unrelated roles that mention analytics tools incidentally

Processing all of these vacancies with an LLM would increase:

- API usage
- token consumption
- execution time
- irrelevant records in the downstream pipeline

For this reason, Data Career Copilot uses deterministic filtering as the first relevance gate.

The design follows a simple principle:

> **Use deterministic rules for inexpensive, reproducible filtering and reserve the LLM for tasks that require semantic interpretation.**

---

## 3. Classification Strategy

The filter evaluates multiple signals instead of relying on a single keyword.

The classification process considers information such as:

- job title
- job description
- target-role terminology
- analytical responsibilities
- Data / BI technologies
- automation signals
- non-priority technical domains
- combinations of multiple relevance signals

This is important because job titles alone are unreliable.

A role called `Data Analyst` may clearly belong to the target domain, while a more ambiguous title such as `Functional Analyst` may still contain strong Data and Automation responsibilities.

Conversely, a vacancy may contain words such as `Python`, `SQL`, or `analytics` while its actual core function belongs to another technical discipline.

The filter therefore combines several signals to determine relevance.

---

## 4. Relevance Filter V1

The first version of the filter established the deterministic baseline.

Its logic combined:

### Text normalization

Job titles and descriptions are normalized before evaluating matching rules.

This reduces inconsistencies caused by differences in capitalization, punctuation, accents, or formatting.

### Target-role signals

The filter looks for evidence related to the job families targeted by the project, including:

- Data Analyst
- Business Intelligence
- BI Analyst
- Operations Analytics
- Business Analytics
- Data & Automation roles

### Analytical signals

Descriptions can contribute additional evidence when they contain responsibilities or concepts related to:

- data analysis
- reporting
- dashboards
- KPIs
- SQL
- Power BI
- data transformation
- automation
- business analysis

### Relevance pillars

Rather than depending entirely on individual keywords, the filter groups signals into broader relevance dimensions or **pillars**.

A vacancy supported by multiple independent analytical signals is more reliable than one matching only an isolated keyword.

### Technical blockers

The filter also detects technical domains that are outside the primary scope of the project.

This helps prevent vacancies from being classified as relevant merely because they share technologies such as Python or SQL with Data Analyst roles.

### Relevance score

The signals contribute to a deterministic relevance score.

The score is not a candidate compatibility score.

It represents only the strength of the evidence that the vacancy belongs to one of the target job families.

The final decision is produced through explicit rules rather than an LLM-generated probability.

---

## 5. Validation Methodology

To evaluate the first version of the classifier, a sample of **50 job postings** was manually reviewed and labeled according to whether each vacancy should continue through the Data Career Copilot pipeline.

The manual classification was then compared with the output produced by Relevance Filter V1.

This produced the following confusion matrix:

| | Manually Relevant | Manually Not Relevant |
|---|---:|---:|
| **Filter Relevant** | 44 TP | 4 FP |
| **Filter Not Relevant** | 1 FN | 1 TN |

Where:

- **TP — True Positive:** relevant vacancy correctly retained
- **FP — False Positive:** irrelevant vacancy incorrectly retained
- **FN — False Negative:** relevant vacancy incorrectly rejected
- **TN — True Negative:** irrelevant vacancy correctly rejected

The purpose of this validation was not to establish a production-grade benchmark from a large statistical sample.

Instead, it provided an empirical baseline for identifying classification errors and improving the rules.

---

## 6. Evaluation Metrics

Using the manually labeled 50-job sample, Relevance Filter V1 achieved:

### Precision

```text
Precision = TP / (TP + FP)

Precision = 44 / (44 + 4)

Precision ≈ 91.7%
```

Precision measures how many vacancies classified as relevant were actually relevant according to the manual labels.

### Recall

```text
Recall = TP / (TP + FN)

Recall = 44 / (44 + 1)

Recall ≈ 97.8%
```

Recall is particularly important in this system because rejecting a potentially valuable vacancy too early means that the opportunity never reaches the deeper AI matching stage.

### F1 Score

```text
F1 = 2 × (Precision × Recall) / (Precision + Recall)

F1 ≈ 94.6%
```

Summary:

| Metric | V1 Result |
|---|---:|
| Evaluation sample | 50 jobs |
| True Positives | 44 |
| True Negatives | 1 |
| False Positives | 4 |
| False Negatives | 1 |
| Precision | 91.7% |
| Recall | 97.8% |
| F1 Score | ~94.6% |

These metrics represent the **V1 baseline on this specific manually labeled sample**.

They should not be interpreted as guaranteed production performance or as performance metrics for V1.1.

---

## 7. Error Analysis

The evaluation was useful not only because it produced metrics, but because individual errors revealed weaknesses in the classification strategy.

Three cases were particularly useful during the refinement process.

### Case 1 — Banorte

The vacancy generated limited analytical evidence:

```text
relevance_score: 3
pillars: 1
```

The available signals were not strong enough to justify deeper processing.

This case reinforced the importance of requiring sufficient evidence instead of allowing isolated analytical terminology to determine relevance.

---

### Case 2 — Seguros Monterrey

This vacancy initially appeared related to the target domain because its description contained several Data-related signals.

However, deeper inspection showed that its central function was oriented toward Machine Learning / Data Science rather than the Data & Analytics roles targeted by the project.

This exposed an important weakness:

> A vacancy can contain many Data-related signals and still belong to a non-priority technical function.

V1.1 therefore introduced a functional Machine Learning / Data Science blocker.

Instead of rejecting a vacancy because it contains a single ML-related term, the blocker requires multiple explicit signals indicating that Machine Learning or Data Science is central to the role.

The current rule requires at least **two explicit ML / Data Science functional signals** before activating this blocker.

This reduces the risk of rejecting ordinary analyst positions that simply mention Machine Learning as a desirable skill.

---

### Case 3 — Zincro

Zincro demonstrated the opposite problem.

Its title was closer to an ambiguous `Functional Analyst` profile and could therefore be rejected by an overly strict title-based classifier.

However, the actual responsibilities contained meaningful evidence related to:

- Data
- Automation
- business processes

The vacancy reached:

```text
relevance_score: 5
pillars: 1
hybrid_data_automation_role: true
```

This case showed why ambiguous job titles should not automatically be discarded.

V1.1 introduced a hybrid Data + Automation path capable of retaining these roles when the description provides sufficient supporting evidence.

---

## 8. From V1 to V1.1

The objective of V1.1 was not to redesign the classifier around individual errors.

Instead, the observed errors were used to identify more general classification patterns.

The main changes were:

### 1. Better handling of ambiguous titles

Titles such as:

```text
Functional Analyst
Analista Funcional
```

can represent very different functions depending on the organization.

V1.1 allows the job description to provide additional evidence before rejecting these vacancies.

### 2. Hybrid Data + Automation path

A dedicated path was introduced for vacancies combining analytical and automation responsibilities.

This is relevant because the target job space is not limited to traditional Data Analyst titles.

Some roles combine:

```text
Data
+
Business Processes
+
Automation
```

and may still represent valuable opportunities.

### 3. Functional ML / Data Science blocker

V1.1 distinguishes between:

```text
A vacancy mentioning Machine Learning
```

and:

```text
A vacancy whose core function is Machine Learning / Data Science
```

The blocker requires at least two explicit functional signals before treating the role as primarily ML / Data Science.

### 4. Diagnostic information

V1.1 also exposes diagnostic information that makes filter behavior easier to inspect during testing.

The filter identifies itself as:

```text
Data Relevance Filter V1.1
```

This supports regression testing and future comparisons between filter versions.

---

## 9. Why the Threshold Was Not Increased

A possible reaction to false positives would be to simply increase the minimum relevance score.

That approach was intentionally avoided.

For example, Zincro reached a relevance score of:

```text
5
```

while still representing a legitimate hybrid Data + Automation opportunity.

Increasing the global threshold based only on a small number of false positives could therefore create additional false negatives.

This matters because the cost of the two error types is asymmetric.

### False Positive

An irrelevant vacancy passes the filter.

Cost:

```text
additional downstream processing
```

### False Negative

A potentially valuable vacancy is rejected before candidate matching.

Cost:

```text
potential opportunity permanently lost
```

For this system, maintaining high recall is therefore particularly important.

The preferred strategy is to improve **specific classification rules** when a generalizable error pattern is discovered rather than continuously increasing the global threshold.

---

## 10. Avoiding Overfitting

Rule-based classifiers can easily become overfitted when every incorrect prediction results in another keyword or exception.

For example:

```text
Error
  ↓
Add keyword
  ↓
New error
  ↓
Add exception
  ↓
Add another keyword
```

Eventually, the classifier becomes difficult to maintain and performs well only on the examples used during development.

Data Career Copilot therefore follows a more conservative approach.

A rule should preferably be added when an error reveals a **generalizable semantic pattern**, not merely because a specific company or vacancy was misclassified.

This is why V1.1 introduced concepts such as:

- hybrid Data + Automation roles
- ambiguous functional titles
- functional ML / Data Science detection

rather than company-specific rules.

No company names are required by the production classifier.

---

## 11. Current V1.1 Design

The current filter can be summarized as:

```text
Normalized Vacancy
        ↓
Target Role Signals
        ↓
Analytical / Technical Signals
        ↓
Relevance Pillars
        ↓
Hybrid Data + Automation Detection
        ↓
Non-priority Functional Blockers
        ↓
Deterministic Relevance Rules
        ↓
Relevant / Not Relevant
```

This layer intentionally remains independent from candidate compatibility.

A vacancy may therefore be:

```text
Relevant to Data & Analytics
```

but later receive:

```text
Review
```

or:

```text
Discard
```

during candidate matching.

For example, a relevant vacancy may still conflict with:

- required seniority
- required technologies
- English level
- academic requirements
- geographic restrictions
- work modality

This separation prevents the relevance classifier from becoming responsible for decisions that belong to the candidate-matching layer.

---

## 12. Design Principles

The Relevance Filter follows several principles:

**Deterministic behavior**

The same vacancy should produce the same filtering result when the rules remain unchanged.

**High recall**

Potentially valuable opportunities should not be rejected too aggressively during the first stage.

**Explainability**

Classification decisions should be traceable to observable signals and rules.

**Cost control**

Clearly irrelevant vacancies should be removed before consuming LLM resources.

**Separation of concerns**

Job relevance and candidate compatibility are treated as different problems.

**Controlled iteration**

Rules are changed when error analysis reveals a reusable pattern rather than to correct isolated examples.

---

## 13. Limitations

The current evaluation has several limitations.

### Small labeled dataset

The V1 benchmark contains only 50 manually labeled vacancies.

This is useful for initial validation but insufficient to claim broad classifier performance across the entire labor market.

### Search-source dependency

The distribution of jobs evaluated by the filter depends on the vacancies returned by Google Jobs through SerpApi.

Changes in source composition may affect filter behavior.

### Rule-based language coverage

New job titles or terminology may appear that are not adequately represented by the current rules.

### Ambiguous vacancies

Some positions genuinely combine several disciplines.

Roles spanning Analytics, Automation, Data Engineering, Product, or Machine Learning may not have a single obvious classification.

### V1.1 requires a larger controlled evaluation

The existing Precision, Recall, and F1 metrics belong to the V1 baseline.

V1.1 has been tested during workflow development and regression analysis, but a new labeled benchmark should be executed before publishing comparable V1.1 performance metrics.

---

## 14. Next Validation Steps

Future evaluation should expand the labeled dataset and measure the behavior of V1.1 independently.

A stronger validation process would include:

```text
Larger labeled dataset
        ↓
Balanced relevant / irrelevant examples
        ↓
V1.1 predictions
        ↓
Confusion matrix
        ↓
Precision / Recall / F1
        ↓
Error analysis
        ↓
Regression tests
```

Additional analysis could also segment performance by job family, such as:

- Data Analyst
- Business Intelligence
- Operations Analytics
- Business Analytics
- Data + Automation
- ambiguous analytical roles
- non-priority technical roles

This would make it possible to determine not only how often the classifier is correct, but **where it performs well and where additional refinement is necessary**.

---

## Related Documentation

- [System Architecture](architecture.md)
- [Pipeline Documentation](pipeline.md)
- [Main Project README](../README.md)