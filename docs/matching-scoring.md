# Candidate Matching & Deterministic Scoring

## 1. Purpose

After a vacancy has passed the relevance filter and its requirements have been converted into structured data, Data Career Copilot evaluates how well the opportunity aligns with the candidate profile.

This stage answers a different question from the Relevance Filter:

> **Given that this is a relevant Data & Analytics vacancy, how well does it match the candidate's demonstrated experience, skills, constraints, and career direction?**

The matching system uses a hybrid architecture:

```text
Structured Vacancy
        +
Candidate Profile
        ↓
Semantic Evaluation with LLM
        ↓
Structured Compatibility Dimensions
        ↓
Deterministic Scoring Engine
        ↓
Compatibility Score
        ↓
Apply / Review / Discard
```

The LLM does **not** generate the final numerical compatibility score.

Its responsibility is to interpret the vacancy and candidate profile semantically.

The numerical score is calculated afterward using deterministic JavaScript rules.

---

## 2. Why Separate Semantic Matching from Scoring?

Candidate-to-job matching contains two different types of problems.

### Semantic interpretation

Some requirements cannot be evaluated reliably through exact keyword matching.

For example:

```text
Vacancy:
"Develop operational dashboards and monitor business KPIs."

Candidate:
"Experience managing operational metrics and reporting through
Power BI, Looker Studio and spreadsheets."
```

The wording is different, but the underlying experience may be related.

This is where an LLM is useful.

### Numerical decision logic

Once compatibility has been interpreted across predefined dimensions, the system still needs to decide:

- how important each dimension is
- how missing requirements should affect the score
- which conditions should cap the maximum score
- how the final recommendation should be classified

These decisions should remain reproducible.

For this reason:

> **The LLM interprets evidence; deterministic rules calculate the score.**

This prevents the model from arbitrarily deciding that one vacancy is, for example, an 87% match and another is an 82% match without a stable scoring framework.

---

## 3. Candidate Profile

The candidate profile is stored independently from the workflow logic in Google Sheets.

The sheet uses a simple structure:

```text
campo | valor
```

The workflow converts these rows into a structured candidate object before matching.

The profile contains information such as:

- target roles
- total professional experience
- relevant operational and analytical experience
- tools with demonstrated proficiency
- tools with intermediate proficiency
- English level
- salary expectations
- preferred work modalities
- geographic constraints
- professional restrictions
- CV content

Separating the candidate profile from the workflow provides an important architectural benefit:

> Candidate information can evolve without rewriting the matching logic.

---

## 4. Evidence-Based Matching

The matching prompt is designed to evaluate only information supported by the vacancy and candidate profile.

It avoids several common shortcuts.

### Total experience is not Data experience

The candidate has more than eight years of professional experience.

However:

```text
8+ years of professional experience
≠
8+ years of Data experience
```

The system therefore distinguishes between:

- total professional experience
- analytical experience
- technical Data experience

This prevents professional seniority from being incorrectly converted into technical seniority.

### Portfolio projects are valid evidence

Projects can demonstrate practical experience with technologies such as:

- Python
- SQL
- PostgreSQL
- Power BI
- automation tools

However, portfolio experience is not automatically treated as equivalent to years of professional employment using those technologies.

### Skill level matters

A technology being present in the candidate profile does not automatically mean advanced expertise.

For example:

```text
Python — Intermediate
```

should not satisfy a requirement such as:

```text
5+ years of advanced Python development
```

The matching system therefore considers both the technology and the demonstrated level.

---

## 5. Compatibility Dimensions

The LLM evaluates the vacancy across eight predefined dimensions.

### 1. Related Experience

Evaluates how closely the candidate's demonstrated experience aligns with the experience required by the vacancy.

Examples include:

- Data analysis
- operational analytics
- KPI management
- reporting
- process analysis
- business analysis
- automation

### 2. Tools & Technologies

Evaluates alignment between required technologies and the candidate's demonstrated toolset.

This can include:

- Python
- SQL
- PostgreSQL
- Power BI
- Looker Studio
- Excel
- automation platforms
- APIs
- related technical tools

### 3. Responsibilities

Compares the actual responsibilities of the role with activities the candidate has previously performed or demonstrated through projects.

### 4. Industry / Context

Evaluates whether the candidate has experience in a related operational or business environment.

Industry similarity can be useful but is not treated as equivalent to technical competency.

### 5. Seniority

Evaluates the level of experience, autonomy, complexity and leadership expected by the vacancy.

The system does not infer seniority solely from total years of professional experience.

### 6. Language

Evaluates explicit language requirements.

If the vacancy does not specify a language requirement, the system does not invent one.

### 7. Modality & Location

Evaluates whether the work arrangement is compatible with the candidate's geographic constraints.

The candidate can work:

```text
CDMX
Estado de México
Remote from Mexico
```

The candidate cannot relocate outside CDMX or Estado de México.

Therefore:

```text
On-site outside CDMX / Estado de México
→ Not compatible

Hybrid requiring attendance outside CDMX / Estado de México
→ Not compatible

Remote and executable from Mexico
→ Compatible

Location outside the allowed area but modality unspecified
→ Partial / uncertain
```

### 8. Salary

Evaluates salary compatibility when compensation information is available.

If the vacancy does not disclose compensation, the system does not assume that the salary requirement is satisfied.

---

## 6. Structured LLM Output

Instead of accepting free-form model responses, the workflow uses a Structured Output Parser.

The LLM returns compatibility information in a predefined schema.

In addition to the eight dimensions, the structured evaluation includes fields such as:

```text
strengths
missing_skills
recommendation_reason
critical_requirements_missing
core_technologies_missing
has_exclusionary_requirement
exclusionary_requirement_reason
```

This makes the model output easier to validate and safer to consume programmatically.

The LLM is explicitly instructed **not to produce**:

- compatibility percentages
- numerical scores
- final scoring points
- autonomous application decisions

Those responsibilities belong to the deterministic scoring layer.

---

## 7. Mandatory, Desirable and Core Requirements

Not all requirements have the same importance.

The matching process distinguishes between:

### Mandatory requirements

Requirements explicitly necessary for the position.

Missing a mandatory requirement may materially reduce compatibility.

### Desirable requirements

Skills or experience that improve fit but are not necessarily required.

Missing a desirable requirement should not be treated the same way as missing a mandatory one.

### Core technologies

Technologies central to performing the role.

Examples could include:

```text
SQL
Power BI
Tableau
Python
```

depending on the specific vacancy.

Missing multiple core technologies can significantly limit realistic compatibility even when other dimensions are strong.

---

## 8. Alternative Technology Requirements

Vacancies frequently describe requirements using alternatives.

Examples:

```text
Power BI or Tableau

Python / R

Experience with tools such as Power BI, Tableau, Qlik or similar
```

These should not be interpreted as requiring every technology listed.

The matching logic treats explicit alternative groups differently from independent requirements.

If the candidate sufficiently demonstrates one accepted alternative, the technology group may be considered covered.

For example:

```text
Vacancy:
Power BI or Tableau

Candidate:
Power BI
```

should not generate a missing-core-technology penalty solely because Tableau is absent.

This rule applies only to genuine alternatives.

It does not override separate requirements related to:

- years of experience
- proficiency level
- certifications
- other independently mandatory technologies

---

## 9. Exclusionary Requirements

Some requirements can make a vacancy incompatible regardless of otherwise strong alignment.

Examples can include:

- mandatory academic credentials
- explicit advanced language requirements
- incompatible geographic requirements
- mandatory technical conditions clearly absent from the profile

The system marks an exclusionary requirement only when:

1. the vacancy clearly states the condition, and
2. the candidate profile provides sufficient evidence that the condition is not satisfied.

Missing information alone is not automatically treated as proof of incompatibility.

This distinction prevents the system from rejecting vacancies based on assumptions.

---

## 10. Deterministic Scoring Model

After the LLM produces the structured compatibility evaluation, JavaScript calculates the numerical score.

The scoring model uses the following weights:

| Dimension | Maximum Weight |
|---|---:|
| Related Experience | 25 |
| Tools & Technologies | 25 |
| Responsibilities | 15 |
| Industry / Context | 10 |
| Seniority | 10 |
| Language | 5 |
| Modality & Location | 5 |
| Salary | 5 |
| **Total** | **100** |

The largest weights are assigned to:

```text
Related Experience
+
Tools & Technologies
```

Together, these dimensions represent 50% of the base score.

This reflects the project's objective of prioritizing vacancies where the candidate can demonstrate meaningful practical alignment.

---

## 11. Base Score

The system first calculates a base compatibility score from the eight weighted dimensions.

This value is stored separately as:

```text
score_base
```

Keeping the base score separate is important because the final score may later be affected by:

- missing critical requirements
- missing core technologies
- exclusionary conditions
- compatibility caps

This makes the scoring process more explainable.

For example:

```text
Base Score
    ↓
Penalties
    ↓
Compatibility Caps
    ↓
Final Score
```

A high semantic match therefore does not automatically guarantee a high final score.

---

## 12. Critical Requirement Penalty

Missing critical requirements reduce the score.

The penalty is calculated as:

```text
3 points × number of critical requirements missing
```

with a maximum penalty of:

```text
12 points
```

Conceptually:

```text
critical_penalty = min(missing_critical_requirements × 3, 12)
```

This prevents an unlimited penalty while still making mandatory gaps visible in the final result.

---

## 13. Core Technology Caps

Missing technologies that are central to the role can limit the maximum possible compatibility score.

The current caps are:

| Missing Core Technologies | Maximum Final Score |
|---|---:|
| 0 | No additional cap |
| 1 | 79 |
| 2 | 69 |
| 3 or more | 59 |

This rule prevents a vacancy from receiving an unrealistically high recommendation simply because the candidate performs well across softer dimensions.

For example:

```text
Strong responsibilities match
Strong industry match
Compatible location
Compatible salary
```

cannot fully compensate for several mandatory technologies that are central to performing the job.

---

## 14. Additional Compatibility Caps

The system also contains safeguards for other major incompatibilities.

### Exclusionary requirement

If a confirmed exclusionary requirement exists:

```text
Maximum final score = 59
```

### Low or insufficient tool compatibility

If tool compatibility is materially insufficient:

```text
Maximum final score = 79
```

These caps are applied after the base compatibility evaluation.

The objective is to prevent additive scoring from hiding structural incompatibilities.

---

## 15. Why Use Caps Instead of Only Penalties?

Purely additive scoring can create misleading results.

Consider a simplified example:

```text
Experience        25/25
Responsibilities  15/15
Industry          10/10
Seniority         10/10
Language           5/5
Location           5/5
Salary             5/5
Tools              5/25
```

Even with poor technical alignment, the vacancy could accumulate a relatively strong total from the remaining dimensions.

A compatibility cap introduces a business rule:

> Some gaps are structurally important enough that strength in unrelated dimensions should not fully compensate for them.

This makes the score closer to the actual decision logic used when evaluating whether a vacancy deserves attention.

---

## 16. Final Recommendation

After penalties and caps are applied, the final compatibility score is converted into one of three recommendation categories.

| Final Score | Recommendation |
|---|---|
| 80–100 | Apply |
| 60–79 | Review |
| 0–59 | Discard |

### Apply

The vacancy demonstrates strong overall compatibility and no structural gap reduces it below the application threshold.

### Review

The vacancy contains meaningful alignment but also uncertainty or gaps that justify manual inspection.

### Discard

The vacancy contains substantial incompatibilities or confirmed constraints that make it a low-priority opportunity.

These labels are decision-support categories.

They do not autonomously submit applications or make career decisions for the candidate.

---

## 17. Example Decision Flow

A vacancy can initially show strong compatibility:

```text
Semantic Evaluation
        ↓
Strong Experience Match
Strong Responsibility Match
Strong Industry Match
Compatible Location
        ↓
Base Score: High
```

But suppose the vacancy requires two core technologies not demonstrated by the candidate.

The deterministic layer applies:

```text
2 Core Technologies Missing
        ↓
Maximum Score = 69
        ↓
Recommendation = Review
```

This is one of the main reasons the project separates semantic evaluation from deterministic scoring.

The LLM can recognize transferable experience, while the scoring engine ensures that important technical gaps remain visible.

---

## 18. Validation Cases

Several real vacancies processed during development were used as regression examples for the matching and scoring logic.

### Kuna Capital

Observed result:

```text
Final Score: 97
Recommendation: Apply
```

This served as a strong positive-control case.

### YouTube

Observed result:

```text
Final Score: 72
Recommendation: Review
```

The vacancy demonstrated meaningful alignment but did not satisfy enough conditions to reach the Apply threshold.

### CS Servicios

Observed result:

```text
Final Score: 69
Recommendation: Review
```

### Ultimate Jet

Observed result:

```text
Final Score: 59
Recommendation: Discard
```

Important gaps included requirements related to:

- English
- Tableau
- academic background

This case was useful for validating the interaction between otherwise relevant experience and structural requirements.

### Zincro

Observed result:

```text
Final Score: 59
Recommendation: Discard
```

The role showed strong Data + Automation relevance, but geographic compatibility reduced its viability.

This illustrates an important distinction:

```text
Relevant vacancy
≠
Compatible vacancy
```

A vacancy can correctly pass the Relevance Filter and still be discarded later because candidate-specific constraints are evaluated in a different stage.

---

## 19. Full-Run Observed Results

During one validated full execution, the workflow processed:

```text
47 new vacancies
```

The matching and scoring system classified them as:

| Recommendation | Vacancies |
|---|---:|
| Apply | 9 |
| Review | 7 |
| Discard | 31 |
| **Total** | **47** |

The notification layer surfaced:

```text
9 Apply
+
7 Review
=
16 prioritized opportunities
```

while the 31 Discard results remained recorded but were not expanded into full recommendation cards in the email summary.

These values represent **one validated workflow execution**.

They should not be interpreted as expected proportions for future job-search runs.

---

## 20. Explainability

The system preserves several intermediate outputs to make the final recommendation easier to understand.

These include:

```text
base score
final score
strengths
critical gaps
missing core technologies
recommendation reason
exclusionary conditions
```

This is preferable to returning only:

```text
87% Match
```

without explaining how that value was produced.

The architecture therefore prioritizes:

> **Structured evidence + deterministic decision logic + traceable output**

over opaque AI-generated scoring.

---

## 21. Design Principles

The matching system follows several principles.

**LLM for semantic interpretation**

Natural-language requirements and transferable experience require contextual reasoning.

**Code for numerical decisions**

Weights, penalties, thresholds and caps remain deterministic.

**Evidence over assumptions**

The system should not invent experience, language proficiency, academic credentials or relocation willingness.

**Skill level matters**

Knowing a technology is not equivalent to demonstrating advanced professional experience with it.

**Constraints matter**

Location, language, academic requirements and core technologies can materially affect compatibility.

**Relevance and compatibility are separate**

A vacancy can belong to the correct job family while still being a poor candidate match.

**Human decision remains final**

Apply / Review / Discard prioritizes attention; it does not replace the candidate's final decision.

---

## 22. Limitations

The current matching system still has limitations.

### LLM interpretation variability

Structured outputs reduce variability but do not eliminate differences in semantic interpretation.

### Candidate profile quality

The system can only evaluate evidence represented in the candidate profile.

Incomplete or outdated profile information can affect matching quality.

### Vacancy quality

Job descriptions may omit important information such as:

- salary
- work modality
- language requirements
- seniority
- exact technology expectations

### Rule calibration

The current weights, penalties and thresholds were designed for the project's target roles and validated through practical cases.

They are not universal job-matching constants.

### Limited outcome feedback

The current system evaluates compatibility before application.

It does not yet use downstream outcomes such as:

```text
Application
Interview
Technical Interview
Offer
Rejection
```

to recalibrate the scoring model.

---

## 23. Future Improvements

Future versions could introduce an outcome-based feedback loop.

For example:

```text
Vacancy Score
      ↓
Application
      ↓
Recruiter Response
      ↓
Interview
      ↓
Outcome
      ↓
Historical Matching Analysis
```

Once sufficient real application data exists, it would become possible to analyze questions such as:

- Which compatibility dimensions correlate with recruiter responses?
- Are the current weights calibrated appropriately?
- Which missing skills most frequently block progression?
- Does the Apply threshold identify higher-conversion opportunities?
- Which job families produce the strongest outcomes?

This would allow the scoring framework to evolve using observed career outcomes rather than intuition alone.

---

## Related Documentation

- [System Architecture](architecture.md)
- [Pipeline Documentation](pipeline.md)
- [Relevance Filter](relevance-filter.md)
- [Main Project README](../README.md)