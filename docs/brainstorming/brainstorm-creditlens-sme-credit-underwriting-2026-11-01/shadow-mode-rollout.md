# Shadow Mode Rollout Plan — CreditLens SME Credit Underwriting
**Version:** 1.0  
**Date:** 2026-11-01  
**Audience:** Implementation engineer + bank relationship manager reading together

---

## What This Document Is

This is an implementation guide, not a design document. It tells you what to build, what to collect, what to show, when to move, and who decides. It assumes the session finding verbatim: shadow mode is not a testing phase — it is the trust-building protocol. If it is cut under time pressure, the adoption strategy collapses. The timeline is six months minimum. This document explains why and gives you the arguments to defend it.

---

## Prerequisites: What Must Be True Before Shadow Mode Begins

Shadow mode does not start on the day the model is deployed. It starts when two conditions are met. Both are non-negotiable.

### Prerequisite 1 — Data Access Agreement Signed

The first move is not technical. It is one signature.

A senior bank officer (Chief Risk Officer or equivalent) must sign a data access agreement that defines:

- Which transaction fields CreditLens is authorized to read, by name and data type
- The format and frequency of the data feed (batch nightly vs. near-real-time event stream)
- Under whose authorization the feed runs (named individual, not a department)
- What CreditLens may not do with the data (no model training on overridden decisions as ground truth, no data leaving the bank perimeter)
- The retention period for shadow mode outputs and who owns them

Without this document, the engineer has no stable schema. The pipeline breaks when field names change. The loan officer has no legal basis to act on any finding. The bank's legal team kills the project in procurement review the moment a question about data governance surfaces without a signed answer.

**Milestone PRE-1:** Data access agreement signed, countersigned by CreditLens technical lead, filed in the project audit folder. Date recorded. No shadow mode work begins before this date.

### Prerequisite 2 — Clean, Stable Feature Pipeline

Before shadow mode begins, the feature pipeline must pass a stability gate. This is not an internal quality bar. It is a precondition for meaningful results.

A model that is wrong in random ways — because features are noisy, schema mismatches corrupt values, or the pipeline silently drops records — produces shadow mode outputs that are unintelligible to a loan officer. The officer sees the model wrong on cases it should get right, concludes the model is garbage, and the trust window closes permanently. You do not get a second chance.

**Stability gate criteria (all must pass for 14 consecutive days before shadow mode opens):**

| Check | Threshold |
|---|---|
| Feature completeness rate (non-null core features per record) | ≥ 95% |
| Schema drift alerts in the last 14 days | 0 |
| Pipeline run success rate | ≥ 99% |
| PSI on each core feature vs. baseline | < 0.1 |
| Score distribution vs. holdout calibration set (KS stat) | < 0.05 |

**Milestone PRE-2:** Stability gate report generated, reviewed by ML engineer and risk manager jointly, both sign off. Report filed. Shadow mode clock starts the next business day.

---

## Phase 1 — Instrument and Observe (Months 1–2)

**Goal:** Collect the first complete cohort of parallel decisions without surfacing results to officers yet.

### What Happens

Every loan application processed through the existing manual workflow is also scored by the CreditLens model in parallel. The model's output — score, band (low/medium/high risk), SHAP values per feature — is written to the shadow log. The officer makes their decision independently. No CreditLens output appears in the officer's interface.

**What is collected per application in the shadow log:**

```
application_id          string
application_date        timestamp
officer_id              string (hashed, not name — PII-safe for analytics)
officer_decision        approve | decline | refer
officer_decision_time   timestamp
model_score             float (0–1)
model_band              low | medium | high
model_recommendation    approve | decline
shap_top_5              JSON array: [{feature, value, shap_contribution}]
feature_snapshot        JSON: all features at decision time
model_version_hash      string (immutable — which model produced this)
threshold_version_hash  string (which threshold was active)
shadow_cohort_id        string (groups this record into a reporting cohort)
```

The officer identity map — the table linking each pseudonymous officer_id to real officer identity — lives in PostgreSQL in a dedicated, access-controlled schema. Application and analytics roles have no read grant on it; access is limited to the risk_manager and dba roles. De-anonymisation (resolving an officer_id back to a named officer) can only be triggered by a user holding the risk_manager role, and every invocation is written to the audit log with the requesting user, target identifier, and timestamp. The application_id is treated as pseudonymous data under the same access controls as officer_id: it is excluded from logs, analytics exports, and third-party integrations, and reads require the same privileged role as the identity map. In practice, analysts and downstream services operate only on these pseudonymous identifiers, so re-identification is a deliberate, audited break-glass action rather than a routine query.


Nothing in this log is shown to the officer. Nothing is written to the case management system. The shadow log is append-only. It cannot be edited or deleted.

### Infrastructure Checklist for Phase 1

- [ ] Shadow scoring service deployed, receiving loan events via RabbitMQ
- [ ] Shadow log database created (append-only, with audit trigger on any attempted DELETE or UPDATE)
- [ ] Model version hash and threshold version hash embedded in every log record at write time
- [ ] Officer decision captured from existing workflow (webhook or polling — confirm with bank IT)
- [ ] Data feed from bank transaction system confirmed stable (see PRE-2 gate)
- [ ] Keycloak role `shadow_reader` created — only risk manager and ML engineer have it; no loan officers

**Milestone 1.1:** First 30 applications logged with complete data. ML engineer reviews log manually. Any anomaly (missing fields, null SHAP, schema mismatch) is treated as a blocker and fixed before proceeding.

**Milestone 1.2:** First 200 applications logged. Completeness rate confirmed above 95%. Phase 2 begins.

---

## Phase 2 — Internal Review (Months 2–4)

**Goal:** Build the internal evidence base before exposing results to officers. Catch model blind spots early. Confirm the model is worth showing.

### What Happens

The risk manager and ML engineer review shadow mode outputs weekly in a private session. They are looking for:

1. **Systematic disagreements** — segments where the model consistently recommends the opposite of the officer. These are hypotheses, not conclusions. Each one gets a note in the review log: "model more conservative on sector X" or "model more aggressive on applications with < 12 months trading history."

2. **Directional agreement rate** — what percentage of cases did model and officer agree? This number is not a success metric yet. It is a calibration reference. If it is above 90% from month one, the model is probably scoring the easy cases and not being tested on hard ones. That is a problem too.

3. **Early repayment signal** — for any applications from 6+ months prior that are now in the repayment phase, is the model's original shadow score correlated with repayment behavior? This is the first real evidence. It is only available if the bank has repayment data connected to application IDs. Confirm with bank IT during Phase 1.

**Nothing from Phase 2 is shared with loan officers.** Phase 2 is the period in which you make sure the model is worth showing. If you find a systematic problem in the feature pipeline or a model blind spot affecting a whole sector, you fix it before Phase 3. If you fix the model, Phase 1 collection restarts for the affected cohort. The clock does not continue on corrupted data.

**Milestone 2.1:** First internal review session completed. Review log entry written: disagreement rate, top 3 disagreement patterns, any pipeline issues found.

**Milestone 2.2:** 100 applications with 60+ days repayment data available. First correlation analysis between model score and repayment stress run. Results documented. If correlation is directionally correct (higher model risk score → higher repayment stress rate), proceed. If not, stop and diagnose before Phase 3.

---

## Phase 3 — Officer-Facing Reports (Months 3–6)

**Goal:** Begin showing results to loan officers. Build the evidence the officer needs to form private trust in the model.

### What Happens

At the end of every four-week cycle, each loan officer receives a Shadow Mode Report (see template below). This report covers the last 30 closed applications from their portfolio where both an officer decision and a model recommendation exist in the shadow log.

**The report is honest about where the model was wrong, not just where it was right.** This is the single most important design decision in Phase 3. An officer who receives only "look how often the model agreed with you" will conclude the model is flattery software. An officer who receives "here are 4 cases the model flagged that are now showing repayment stress, and here are 3 cases where the model was wrong and you were right" will start to think about what the model sees that they missed — and what they see that the model misses. That is the beginning of productive collaboration. That is what trust looks like before it becomes habit.

### Who Receives the Report

Reports go to individual loan officers, not to the branch or to management. The report shows only the officer's own cases. An officer never sees another officer's results. The risk manager receives an aggregate version (see section below on governance).

The report is delivered as a PDF generated from the shadow log. It is not interactive in Phase 3. An interactive dashboard is a Phase 4 consideration after graduation from shadow mode.

### Frequency

Every four weeks. Not monthly (calendar month creates uneven cohort sizes). Four-week cycles from the date shadow mode opened.

---

## Shadow Mode Reporting Template

This is the exact format of the report shown to loan officers at four-week intervals.

---

**CreditLens Shadow Review — [Officer Name]**  
**Period:** [Start Date] to [End Date]  
**Applications reviewed this period:** [N]  
**Your decisions covered:** [N approve] approved, [N decline] declined  
**Model had a recommendation for:** [N] of those (the rest lacked sufficient data — see "unscoreable" section)

---

### Section 1 — Where We Agreed

[N] cases: both you and the model recommended the same outcome.

Of these, [N] have 60+ days repayment data available. Repayment performance so far:

- [N] on track (no missed payments)
- [N] showing early stress (one or more late payments)
- [N] not yet assessable (too early)

*This section tells you nothing about whether either of us was right — agreement is not accuracy. It tells you the base rate of agreement so you can read the disagreement section in context.*

---

### Section 2 — Where We Disagreed

[N] cases: you and the model recommended differently.

**Cases where you approved and the model flagged risk:**

| Case # | Your decision | Model band | Key feature the model weighted | Repayment status (if available) |
|---|---|---|---|---|
| [ID] | Approve | High risk | Cash flow volatility last 6 months | [on track / early stress / too early] |
| … | | | | |

**Cases where you declined and the model saw lower risk:**

| Case # | Your decision | Model band | Key feature the model weighted | Notes |
|---|---|---|---|---|
| [ID] | Decline | Low risk | 18-month stable revenue trend | [any context you can note] |
| … | | | | |

*These cases are not accusations. They are the starting point for a conversation about what each of us sees. The model saw a feature pattern. You saw something else. One of us was closer to correct. We will know more as repayment data matures.*

---

### Section 3 — Where the Model Was Wrong (Confirmed)

Cases from previous periods where repayment data has now matured enough to assess.

**Model recommended approve, application is showing repayment stress:**

| Case # | Model's original band | Your original decision | What the model missed |
|---|---|---|---|
| [ID] | Low risk | Approve | [narrative from risk manager review] |

**Model recommended decline, application is performing well:**

| Case # | Model's original band | Your original decision | What the model missed |
|---|---|---|---|
| [ID] | High risk | Decline | [narrative from risk manager review] |

*This section exists because a model that only tells you when it was right is not a tool — it is a marketing brochure. These are the cases we are learning from.*

---

### Section 4 — Unscoreable Cases

[N] applications this period where the model returned "insufficient data" rather than a score.

Common reasons this period:
- [e.g., "fewer than 6 months of connected transaction history"]
- [e.g., "sector classification missing from application form"]

*These are not errors. An "unscoreable" output means the model did not have enough data to form a defensible opinion. The manual underwriting checklist applies to these cases. We are tracking what data would have resolved them to inform the data access roadmap.*

---

### Section 5 — Your Override Rate (If Applicable — Phase 4 Only)

*This section is blank in Phase 3. It becomes active only after graduation from shadow mode when overrides are recorded.*

---

**Questions about this report?** Contact [risk manager name] at [internal contact]. This report is for your use only and is not shared with your manager or branch head.

---

*(End of report template)*

---

## Success Criteria for Graduating from Shadow Mode

Graduation means the model begins appearing in the loan officer's interface as an advisory input. The officer sees the score and SHAP summary before making their decision. Overrides are logged with reason codes. The full accountability architecture is live.

Graduation requires ALL of the following. These are not targets to optimize toward. They are gates. All must pass.

| Criterion | Threshold | Evidence Required |
|---|---|---|
| Minimum shadow mode duration | 6 months from PRE-2 milestone | Date record in audit log |
| Repayment correlation confirmed | Model risk band statistically predictive of 90-day repayment stress (p < 0.05) | Analysis report signed by ML engineer and risk manager |
| Officer report delivery rate | Reports delivered on schedule for at least 3 consecutive cycles with no delivery failure | Delivery log |
| Phase 2 review issues resolved | Zero open model blind spots or pipeline issues flagged in Phase 2 review log | Issue tracker closed-item export |
| Officer comprehension check | At least 3 loan officers have reviewed their reports with the risk manager in a 30-minute session and confirmed they understand what the model is showing | Session notes on file |
| Regulatory sign-off | Compliance officer has reviewed shadow mode outputs and confirmed the model's advisory role is documented in a way consistent with applicable banking regulations | Signed compliance memo |
| Risk manager recommendation | Risk manager formally recommends graduation in writing | Written recommendation filed |

**Who decides:** The decision to graduate from shadow mode is made by the Chief Risk Officer, on the written recommendation of the risk manager, after reviewing the evidence package above. The ML engineer provides the technical evidence. The risk manager provides the operational assessment. The CRO makes the call. No other party has the authority to accelerate or approve graduation.

**If any criterion is not met at month 6:** Shadow mode continues. The gap is documented, a remediation plan is written with a specific date, and graduation is revisited at that date.

---

## Duration Rationale: Why 6 Months Minimum

This is the argument you use when stakeholders push to shorten the timeline.

**The repayment data argument (hardest to dismiss):**  
Most SME loans have a 90-day grace period before stress becomes visible in repayment data. A 3-month shadow mode period produces repayment evidence on only the earliest cohort — with sample sizes too small to be statistically meaningful. At 6 months, you have 3–4 rolling cohorts with observable repayment outcomes. This is the minimum evidence base for the risk manager to make a defensible graduation recommendation. Cutting to 3 months means the CRO is approving a live model without confirmed predictive validity. That is a regulatory exposure, not a schedule optimization.

**The officer trust argument (hardest to rush):**  
The session established that trust is formed through private, repeated exposure to honest results over time. One officer receiving one report once does not build trust. An officer receiving four consecutive reports — including one that honestly shows the model was wrong on two cases — begins to form a mental model of when to weight the score heavily and when to apply their own judgment. That calibration takes months. It cannot be compressed by adding more reports in a shorter window because the cases need time to mature into observable outcomes.

**The compliance argument (clearest to legal):**  
Shadow mode is the evidence of responsible deployment. In the event of a challenged credit decision after go-live, the bank's legal defense includes the shadow mode record: "we ran the model in parallel for 6 months, confirmed it was directionally predictive, and trained our officers on its outputs before relying on it." A 6-week shadow mode period does not produce that defense. The timeline is not an engineering choice. It is a legal record.

**The counter-argument to "we can shorten it if the results look good early":**  
Early results looking good is not evidence of long-term validity. It is evidence of performance on the easy cohort. Hard cases — sector downturns, unusual business models, seasonal volatility — require time to appear in the pipeline. A model that looks excellent at month 2 on the early cohort may fail silently on the harder cases that arrive in months 4 and 5. The graduation criteria require statistical significance on repayment outcomes. That number cannot be faked by looking at it earlier.

---

## Preventing Compliance Theater

The session identified compliance theater as the same problem as shadow mode failure, from the opposite end: both occur when the system shows people what they already knew, confirms their existing behavior, and produces no new information. The rollout must be designed to prevent this.

**Risk: Shadow mode becomes a checkbox.** Reports are generated, delivered, unread. No one asks questions. The model runs for 6 months. Graduation is approved because the calendar said so, not because the criteria were met.

**How the design prevents this:**

1. **Section 3 of the report is mandatory.** Reports with no "where the model was wrong" content are flagged for review. A model that has never been wrong in 6 months of shadow mode is either not being tested on hard cases or is not being reported on honestly. The ML engineer must confirm Section 3 is populated from real data, not omitted because there were no confirmed errors.

2. **The 3-session officer comprehension check is not optional.** At least 3 officers must sit with the risk manager and discuss their reports. This creates a record that the reports were actually read and understood. Sessions are documented with notes on what questions the officer asked. "No questions" is not a valid outcome — the risk manager is expected to prompt: "Tell me about one disagreement from your last report that surprised you."

3. **The graduation evidence package is reviewed by the CRO, not delegated.** The CRO reading the package is the accountability moment. If the package does not contain confirmed repayment correlation data, the CRO cannot sign off. This is not a procedural formality — it is the one moment in the process where executive authority is used to enforce the standard rather than to override it.

4. **The risk manager's recommendation is signed and filed.** It cannot be verbal. It cannot be implicit. It cannot be "we're past 6 months so I assume we're good." It is a written statement that says: here is the evidence, here is my assessment, here is my recommendation. The risk manager owns this professionally.

5. **Override analytics cannot begin until after graduation.** If override logging begins during shadow mode, officers will be aware they are being tracked and will perform compliance rather than make real decisions. Shadow mode and override analytics are sequential, not concurrent. The trust must be established before the accountability layer is visible.

---

## Governance Summary

| Decision | Who Decides | Evidence Required | Escalation Path |
|---|---|---|---|
| Shadow mode may begin | Risk manager + ML engineer | PRE-1 and PRE-2 milestones complete | CRO informed, not required to approve |
| Shadow mode continues past month 6 | Risk manager | Written gap assessment with remediation date | CRO notified if extension > 2 months |
| Graduation to production | Chief Risk Officer | Full evidence package (see graduation criteria table) | No escalation — CRO is the final authority |
| Threshold change during shadow mode | Risk manager approves, ML engineer proposes | Written rationale, old and new threshold logged | Any threshold change resets the shadow cohort clock for affected records |
| Model version change during shadow mode | ML engineer implements, risk manager approves | Version hash logged in all new records; old cohort records retain old hash | Evaluation is run separately per version — versions are not mixed in graduation evidence |

---

## Appendix — Shadow Mode Failure Modes and Mitigations

| Failure Mode | Signal | Mitigation |
|---|---|---|
| Features don't match reality officer sees | Officer consistently notes "the model doesn't see what I see in the business" | Return to Phase 1; audit feature pipeline against what manual underwriting actually looks at; add missing qualitative inputs to data request |
| Reports not being read | Officer cannot describe any disagreement from last report in comprehension session | Escalate to branch manager (not as punishment — as a resourcing problem); simplify report format; reduce to top 5 cases rather than full cohort |
| Sample size too small for significance | Fewer than 15 applications per officer per period | Pool across officers for population-level analysis; individual officer reports note "small cohort — population analysis on file with risk manager" |
| Repayment data not connected to application IDs | Section 3 cannot be populated | This is a data access agreement gap; reopen with bank IT immediately; shadow mode cannot graduate without this connection |
| Stakeholder pressure to skip graduation criteria | Risk manager receives informal instruction to approve graduation | Written request required; risk manager must document the instruction and the evidence gap in writing; CRO must be informed in writing before any graduation under incomplete criteria |
