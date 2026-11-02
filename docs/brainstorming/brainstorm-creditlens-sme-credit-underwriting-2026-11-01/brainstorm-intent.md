# CreditLens — Brainstorm Intent
*Intelligent SME Credit Underwriting Platform*
*Session date: 2026-11-01 · Mode: Facilitator · Techniques: Job to Be Done, Starbursting, Disney Method, Backcasting*

---

## Core value proposition (one sentence)

CreditLens closes the gap between a credit decision being made and that decision being explainable — giving every stakeholder a defensible, auditable, role-appropriate view of why a loan was approved or rejected.

---

## The problem (validated)

Tunisian banks cannot profitably serve 60% of SMEs because they lack:
1. Data infrastructure to assess creditworthiness without formal documentation
2. Knowledge retrieval tools to apply complex BCT/Basel regulations efficiently
3. Monitoring systems to detect deteriorating loan health before default

The actual pain at the human level:
- **Loan officer:** 12 career risks on their desk every Monday — no audit trail, no defensible evidence if a loan defaults
- **SME applicant:** Has a real business with real cash flow that no bank has ever assessed — needs to feel heard and assessed fairly, not just rejected
- **Risk manager:** Nightmare is discovering a cluster of correlated failures that were predictable — signal was in the data all along, nobody was watching

---

## Critical design principles (non-negotiable)

**1. Explainability is a legal obligation, not a feature**
SHAP values are the bank's required disclosure under BCT data protection rules. Every credit decision must include a plain-language summary of the factors that drove it. This is not optional and must be designed in from day one.

**2. The model is always advisory, never authoritative**
The human makes the final decision. The model advises. This is a regulatory requirement. The system must make this distinction visible and enforce it architecturally (no auto-approve path).

**3. Immutable decision snapshots**
Every decision is frozen at the moment it is made: model version hash, feature values, SHAP output, active threshold, RAG citations retrieved, officer override reason (if applicable), all timestamped. Stored in an append-only audit log with no update/delete permissions. The system must be able to reconstruct any decision exactly as it appeared two years later.

**4. Role-keyed complexity rendering**
Keycloak RBAC controls not just access but depth of information:
- Loan officer → plain-language SHAP summary
- Credit analyst → full SHAP waterfall with feature values
- Risk manager → population-level SHAP aggregates + drift metrics
Same underlying explanation, three renders.

**5. Intellectual honesty as a design pattern**
The system must surface uncertainty rather than paper over it:
- Cold-start SME (< 6 months data) → "unscoreable" status, not a low score
- RAG low-confidence retrieval → explicit uncertainty flag, not a confident wrong answer
- Conflicting regulatory circulars → pending state until compliance officer reviews

---

## Scoped build (MoSCoW)

### Must (skeleton — without these the value proposition collapses)
- **Auditability & Accountability (Cluster B)**
  - Immutable decision snapshots (model version + features + SHAP + citations + override reason)
  - Append-only audit log, no admin delete
  - Override logging: mandatory reason code + free text; override analytics pipeline
  - Override rate per officer surfaced to risk manager as bias detection metric
  - Override decisions never ingested as model training ground truth
- **Role-keyed Experience (Cluster D)**
  - RBAC controlling complexity level (plain language / waterfall / portfolio aggregates)
  - Threshold governance: risk manager sets, 4-eyes approval for changes, loan officers read-only
  - Access control: executive overrides blocked, external regulator = read-only export only

### Should (adoption that sticks vs. adoption that collapses)
- **Trust Architecture (Cluster A)**
  - Shadow mode as first deployment phase — model runs parallel to existing decisions, no one acts on it
  - Shadow mode results surfaced to officers after minimum 6 months: "cases where model disagreed with you, outcome was X"
  - Shadow mode designed intentionally; must not be cut under time pressure
- **Build Sequence (Cluster H)**
  - First deliverable is not code — it is a data access agreement with the bank's IT department
  - Feature pipeline must connect to real transaction data before shadow mode begins
  - Features must match the reality the officer sees when reading a file manually

### Could (v2 — require stable trusted core first)
- **SME Dignity & Rights (Cluster F)**
  - v1: read-only SME portal showing features extracted, score, SHAP in plain language, BCT articles checked
  - Downloadable PDF "financial profile" — first time SME sees their own financial fingerprint
  - Proactive outreach: reuse drift monitoring DAG to detect when declined SME's profile crosses threshold; notify assigned officer via RabbitMQ → notification-service (no new infrastructure)
- **Platform as Infrastructure (Cluster G)**
  - PSI drift dashboard framed as macro signal tool for risk manager, not just MLOps health check
  - Post-disbursement alerts routed directly to risk manager, bypassing loan officer (compensation structure workaround)

### Won't (this build)
- **Graceful Degradation edge cases (Cluster C):** Citation frequency dashboard, circular conflict detection, semantic promiscuity detection — require production usage data to tune correctly; document as known limitations
- **Incentive Misalignment (Cluster E):** Compliance theater, ignored post-disbursement monitoring — organizational problems outside software scope; architectural workarounds already included above

---

## Deployment strategy (non-negotiable sequencing)

1. **Signature first** — data access agreement with IT/legal defining fields, format, frequency, authorization
2. **Feature pipeline on real data** — no synthetic data in shadow mode; features must be recognizable to officers
3. **Shadow mode** — minimum 6 months parallel run before production adoption
4. **Honest results** — show officers where the model was right and where it wasn't; don't cherry-pick
5. **Private trust** — adoption happens when one officer makes their own decision to trust the system; it cannot be mandated

---

## Long-horizon vision (out of scope v1, shapes v1 decisions)

**The SME Financial Passport:** SME owns their behavioral financial fingerprint. Portable across banks. The bank's data access agreement (v1) and the SME's ownership right are the same legal question from opposite sides. v1 plants the flag; the passport is the destination.

---

## Key tensions to resolve in product/architecture planning

1. **Override audit quality vs. friction:** Mandatory reasoning produces compliance theater if reason codes are too easy to satisfy. How do you make override reasoning genuinely costly without being so burdensome that officers bypass the tool entirely?

2. **Shadow mode duration vs. time pressure:** 6 months of shadow mode is the right trust-building timeline; project timelines will push to cut it. This tension must be explicitly documented and defended.

3. **RAG confidence vs. coverage:** Low-confidence flag means fewer queries get answered. High-confidence threshold risks surfacing wrong answers confidently. RAGAS evaluation on a real BCT corpus (not toy data) is required before setting the production threshold.

4. **Real transaction data vs. demo requirements:** The platform's core value requires real bank data; demos and thesis defense require something runnable. Synthetic data pipeline must be clearly labeled as such and never used to validate the model's predictive claims.

---

## Positioning (procurement language)

CreditLens is a **decision support layer** that sits alongside the bank's existing process. It does not replace the core banking system, the loan origination workflow, or the document management system. It replaces the Excel scorecard the analyst was using and the PDF search the compliance officer was doing by hand — but that framing is never used in procurement conversations.
