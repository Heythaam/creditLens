# CreditLens — Architecture Decision Records Starter Pack

Generated from brainstorming session: 2026-11-01
Source: `.memlog.md` (Starbursting, Disney Method, Backcasting, Job to Be Done)

---

## ADR-001: Model Output Is Advisory Only — Never Authoritative

**Status:** Proposed

**Context:**
CreditLens produces a credit score and SHAP-based explanation for SME loan applications. There is a regulatory requirement (and a business procurement reality) that no automated system makes a binding lending decision without human sign-off. The loan officer is the accountable party in the regulatory chain, not the model.

**Decision:**
The model output is always advisory. The system presents a score, confidence band, and SHAP explanation to the loan officer as a recommendation. Final approval or rejection is recorded as the officer's decision, not the model's output. The UI, API contracts, and all downstream audit entries will label model output as "recommended" and human action as "decided."

**Consequences:**
- Regulatory compliance is maintained: the system cannot be characterised as an autonomous credit decision engine.
- Procurement objections from bank legal teams are pre-empted — the system is a decision support layer, not a replacement.
- Officer accountability is preserved, which also preserves the audit trail's legal defensibility.
- The model cannot be held solely responsible for a declined application; a human is always in the loop, which is the correct framing for GDPR Article 22 (automated individual decision-making) compliance.
- Shadow mode and override logging become the primary feedback mechanisms because the model has no direct control path.

**Alternatives Rejected:**
- *Straight-through processing for low-risk tiers:* rejected because even a low-risk score creates regulatory exposure if no human reviewed it. The liability offset does not justify the compliance risk.
- *Model-authoritative with human escalation threshold:* rejected for the same reason — the regulatory requirement is categorical, not threshold-based.

---

## ADR-002: Immutable Append-Only Audit Log

**Status:** Proposed

**Context:**
Each credit decision must be fully reconstructable two years later for regulatory audit. This includes the model version hash, feature values at decision time, SHAP output, the active threshold, all RAG citations, and any officer override with its reason code. The integrity of a defensible decision depends on this record never being altered after the fact — including by administrators.

**Decision:**
The audit log is append-only. No UPDATE or DELETE operation is permitted on any audit record by any role, including database administrators and system owners. Records are written once at decision time and are thereafter immutable. The schema enforces this at the database level (insert-only table, no update grants). A separate compliance export path provides read-only access for auditors without exposing write surfaces.

**Consequences:**
- Auditors can reconstruct the exact state of the system at the moment of any decision, two years later or more.
- Executive overrides and officer overrides are recorded but cannot be quietly removed — the audit trail is honest even when the decision was uncomfortable.
- Storage grows monotonically; archival and tiered storage strategy is needed for records beyond the mandatory retention window.
- No soft-delete pattern is available; logical deletion (e.g., marking a record as superseded) must be used if a record needs to be "retired," with the original remaining intact.
- The immutability guarantee must be verified in the deployment runbook — a misconfigured admin role that can issue DELETEs would silently break the compliance promise.

**Alternatives Rejected:**
- *Soft-delete with admin override:* rejected because it creates a surface for after-the-fact manipulation, which destroys the audit trail's legal value.
- *Event sourcing with mutable snapshots:* rejected because snapshot mutability reintroduces the same risk; only a fully append-only event store satisfies the requirement.

---

## ADR-003: Self-Hosted Deployment Only — No SaaS Option

**Status:** Proposed

**Context:**
CreditLens processes SME financial transaction data, behavioral fingerprints, and credit scores. Bank legal and compliance teams require that this data never leave the bank's own perimeter. Any SaaS deployment model — even a private-cloud tenancy — places the data under a third-party's infrastructure and legal jurisdiction, which bank procurement will block before the project reaches production.

**Decision:**
CreditLens is deployed exclusively as a self-hosted system within the bank's own infrastructure (on-premises or bank-controlled private cloud). No telemetry, model training data, feature values, or customer records are transmitted to any external service. The product is shipped as a deployable artifact (container images, Helm charts, infrastructure-as-code) with no mandatory cloud callbacks.

**Consequences:**
- Data sovereignty promises to SMEs and regulators are structurally enforced, not policy-dependent.
- Bank procurement and legal review cycles are shortened because the data governance question is answered architecturally.
- The vendor (CreditLens) has no access to production data for model improvement; feedback loops for retraining must be bank-initiated and bank-controlled.
- Operational burden shifts to the bank's infrastructure team; CreditLens must ship robust runbooks, health checks, and upgrade paths.
- Multi-tenancy (selling to multiple banks) requires independent deployments per bank, not a shared platform.

**Alternatives Rejected:**
- *SaaS with data residency guarantees:* rejected because contractual guarantees do not satisfy bank legal teams in practice — the architecture itself must make the guarantee.
- *Hybrid (metadata in SaaS, data on-prem):* rejected because metadata (feature names, score distributions, model versions) is itself sensitive and subject to the same regulatory constraints.

---

## ADR-004: Shadow Mode Is a Mandatory Pre-Production Phase

**Status:** Proposed

**Context:**
Loan officers have established mental models and existing tools (Excel scorecards, PDF searches). A model that is deployed directly into production will be ignored, gamed, or overridden uniformly until officers trust it. Trust cannot be mandated; it must be earned through demonstrated accuracy on cases the officer already understands the outcome of. The adoption strategy collapses if shadow mode is cut under time pressure.

**Decision:**
Shadow mode is a required phase in every CreditLens deployment. During shadow mode, the model scores every application alongside the existing manual process, but its output is not shown to the officer during the decision. After a defined observation window (recommended minimum: 6 months or 100 scored cases, whichever is longer), a trust report is generated showing cases where the model and the officer disagreed and the subsequent repayment outcome. Shadow mode may not be skipped or shortened below the minimum without written sign-off from the risk manager and a documented rationale.

**Consequences:**
- Officers encounter the model's track record before they are asked to rely on it — trust is evidence-based, not authority-based.
- The trust report ("47 cases you said yes, model said no, 12 showing repayment stress") is the primary adoption artifact; it must be a first-class feature, not a post-hoc report.
- Shadow mode requires clean, stable features from real transaction data — if features are noisy during shadow mode, the trust report will be misleading and adoption will be set back, not advanced.
- The minimum observation window must be defined in the deployment contract, not left to the implementation team's discretion.
- Shadow mode results must not be used to retrain the model (that would be circular); they feed the trust report only.

**Alternatives Rejected:**
- *Pilot with a small loan tier first:* does not build trust in the model for the cases officers care about most (mid-size loans where judgment matters).
- *Parallel run with model output visible but non-binding:* rejected because officers will anchor on the score immediately, which corrupts the baseline; blind shadow is necessary for honest comparison.

---

## ADR-005: Override Logging with Mandatory Reason Code — Never Used as Training Ground Truth

**Status:** Proposed

**Context:**
Loan officers can override the model's recommendation. These overrides carry significant information: they represent cases where human judgment diverged from the model. However, they also carry significant noise: officers may pick the fastest available reason code rather than the accurate one ("compliance theater"). Using overrides as labeled training data would embed officer bias and lazy labeling directly into the next model version.

**Decision:**
Every override requires: (a) a reason code selected from a governed taxonomy, and (b) mandatory free-text reasoning of minimum 20 words. Overrides are logged to the audit trail and routed to the override analytics pipeline for bias detection. They are never ingested as ground-truth labels for model retraining. Model retraining uses only verified repayment outcomes as labels. Override rate per officer is tracked as a bias detection metric; a rate exceeding 40% in a single direction triggers a review flag to the risk manager.

**Consequences:**
- The audit trail captures real reasoning rather than performance of reasoning — the system is designed so that the fastest path is also a documented one.
- Bias detection is structural: the 40% threshold creates an automatic signal for both model failure (segment the model handles poorly) and officer behavior (potential bias problem).
- Overrides feed the override analytics pipeline and the trust report, but not the model — this prevents a feedback loop where officer bias compounds over model generations.
- The free-text requirement increases friction per override, which is intentional: overrides should require justification, not be a frictionless escape hatch.
- The reason code taxonomy must be governed and versioned; stale reason codes that no longer match real scenarios will drive officers back to minimum-effort responses.

**Alternatives Rejected:**
- *Use overrides as weak training signal with downweighting:* rejected because there is no reliable way to distinguish genuine disagreement from lazy labeling in the override record; contamination risk outweighs the signal value.
- *No override logging:* rejected because it destroys the audit trail and eliminates the bias detection signal.

---

## ADR-006: Threshold Governance — Risk Manager Owns, 4-Eyes Control Required

**Status:** Proposed

**Context:**
The score threshold that separates a recommended approval from a recommended decline is a policy parameter, not a model parameter. Changing it directly changes lending outcomes. If an ML engineer can unilaterally adjust the threshold, the credit policy is effectively under engineering control without business accountability. If anyone can adjust it, the integrity of the approval/decline boundary is compromised.

**Decision:**
Score thresholds (approve/decline boundaries, risk tier boundaries) are owned by the risk manager role. Changes require a 4-eyes approval: the ML engineer proposes a threshold change with supporting analytics (score distribution, expected approval rate impact, PSI drift justification), and the risk manager approves or rejects in a governed change record. No threshold change takes effect in production without a signed-off change record. The threshold value in production is stored in a versioned configuration store, not hardcoded or in a feature flag accessible to engineering alone.

**Consequences:**
- Credit policy decisions are made by the person accountable for credit risk, not the person who built the model.
- The change record creates an audit trail for threshold adjustments, which is required for regulatory examination.
- PSI drift alerts become a business tool for the risk manager, not just a DevOps metric — the risk manager can initiate a threshold review in response to portfolio signals.
- Engineering velocity for threshold tuning is deliberately slowed by the approval step; this is correct because threshold changes have direct financial and regulatory consequences.
- The 4-eyes requirement must be enforced in the application layer, not just in policy — the UI must make it impossible to deploy a threshold change without the approval record.

**Alternatives Rejected:**
- *Automated threshold adjustment based on PSI drift:* rejected because automated policy changes without human accountability violate the advisory-only principle and are likely to breach regulatory requirements.
- *Engineering-owned thresholds with risk manager notification:* rejected because notification after the fact is not accountability; the risk manager must be in the approval chain, not the notification chain.

---

## ADR-007: Role-Keyed SHAP Rendering Controlled by RBAC

**Status:** Proposed

**Context:**
SHAP explanations are legally required disclosure. However, the appropriate level of detail differs radically by role: a loan officer needs plain-language guidance they can explain to the SME; a credit analyst needs the full feature waterfall for due diligence; a risk manager needs population-level aggregates for portfolio oversight. Showing the wrong complexity level to a role is both a UX failure and a compliance risk (an officer who cannot understand the SHAP output cannot make an informed decision).

**Decision:**
SHAP output is rendered in three distinct views, keyed by the user's RBAC role in Keycloak: (a) Loan Officer view — plain-language summary of top 3 positive and negative factors, no numerical SHAP values; (b) Credit Analyst view — full SHAP waterfall with feature values and contribution magnitudes; (c) Risk Manager view — population aggregates (SHAP distribution across the current portfolio segment, drift over time). RBAC controls the rendering tier, not just access to the feature. A user cannot self-elevate to a more complex view.

**Consequences:**
- Loan officers receive an explanation they can relay to the SME in plain language, satisfying the disclosure requirement without cognitive overload.
- Analysts have access to the full technical picture for due diligence and model debugging.
- Risk managers see the portfolio-level signal they need for threshold and policy decisions.
- RBAC must be correctly provisioned at deployment; an officer accidentally assigned the analyst role would receive a more complex view than they are trained to interpret.
- The plain-language summary requires a translation layer between raw SHAP values and natural language — this must be maintained as features evolve.
- Adding a new role or a new rendering tier requires a coordinated change to the RBAC configuration, the rendering service, and the access control tests.

**Alternatives Rejected:**
- *Single view with a complexity toggle:* rejected because officers will set the toggle once and forget it, and the appropriate complexity level is role-determined, not preference-determined.
- *Full SHAP waterfall for all roles:* rejected because it fails the disclosure requirement — an explanation that is not understood is not a disclosure.

---

## ADR-008: "Unscoreable" Is a Valid and Required System Output

**Status:** Proposed

**Context:**
Cold-start SMEs — new businesses, informal-sector operators, businesses with fewer than 12 months of transaction history — do not have sufficient data for the model to produce a reliable score. Returning a low score for thin data misrepresents the system's confidence and may cause unjust declines. The system's authority over the scores it does return depends on its honesty about the scores it cannot return.

**Decision:**
"Unscoreable" is a first-class system output, not an error state. When the model's input feature completeness falls below a defined threshold, the system returns an "Unscoreable" result with a structured explanation of which data dimensions are missing and what minimum data would enable scoring. The UI presents this as a graceful handoff to a manual underwriting checklist. The system never interpolates or imputes a score for an application it has flagged as unscoreable. The same principle applies to RAG: a low-confidence retrieval returns a stated low-confidence response with explicit uncertainty, not a fabricated citation.

**Consequences:**
- The system earns epistemic authority by demonstrating intellectual honesty — a system that says "I don't know" when it doesn't know is more trustworthy on the cases where it does commit to a score.
- Cold-start SMEs receive a dignified outcome: they are not declined, they are given a structured path to a scoreable state.
- The manual underwriting checklist for unscoreable cases must be a maintained artifact, not an afterthought — it is the primary UX for a significant segment of first-time applicants.
- The feature completeness threshold that triggers "unscoreable" is a governed parameter, subject to the same 4-eyes control as score thresholds.
- Reporting must distinguish "scored and declined" from "unscoreable" — conflating them would distort approval rate metrics and bias detection.

**Alternatives Rejected:**
- *Return a low score with a low-confidence flag:* rejected because a numerical score invites threshold comparison; "unscoreable" explicitly communicates that no comparison is meaningful.
- *Return a score based on available features only:* rejected because a partial-feature model has unknown calibration on this input distribution; the error bars are too wide to support a defensible decision.

---

## ADR-009: Citations Are Mandatory RAG Output — Uncited Answers Are Blocked

**Status:** Proposed

**Context:**
CreditLens includes a RAG-based compliance assistant that answers questions about regulatory guidance and credit policy. Uncited answers create legal liability: "the AI told me it was compliant" is not a legal defense, and an uncited answer that turns out to be wrong leaves no chain of custody to trace the error. A citation is the compliance layer's equivalent of the audit log — it is the chain of custody for legal reasoning.

**Decision:**
Every answer from the RAG compliance assistant must include at least one citation to a specific indexed regulatory document, identified by document title, section, and version hash. Answers that cannot be grounded in at least one citation are blocked at the API layer — the system returns a structured "no sufficient grounding" response rather than an uncited answer. Citations are displayed to the user with a direct link to the source chunk in the document index. The RAG pipeline is tested against a RAGAS evaluation set that explicitly includes citation precision and recall metrics.

**Consequences:**
- Every compliance answer is traceable to a specific document and version — auditors can verify that the guidance was drawn from the correct regulatory text at the time it was given.
- Hallucinated answers (which would typically have no valid citation) are structurally blocked; the citation requirement acts as a factual grounding check.
- The RAGAS evaluation set must be large enough and diverse enough to catch frequency bias in production (one high-similarity chunk cited in 40%+ of queries is a known failure mode requiring a citation frequency dashboard).
- Users who receive a "no sufficient grounding" response get an honest answer rather than a confident wrong one — consistent with the "unscoreable" epistemology.
- The document index must maintain version hashes for all indexed chunks so that citations remain valid after a regulatory document is updated.

**Alternatives Rejected:**
- *Citations as optional metadata:* rejected because optional metadata will be omitted whenever the retrieval confidence is low — exactly the cases where citation is most important.
- *Citation threshold (cite only if confidence > X):* rejected for the same reason; low-confidence answers are the ones most likely to be wrong and most in need of grounding or blocking.

---

## ADR-010: Circular Conflict Detection Before Indexing New Regulatory Documents

**Status:** Proposed

**Context:**
Regulatory documents are updated over time. A new circular may contradict existing guidance in the index. If the RAG system silently indexes the new document, queries may receive contradictory answers depending on which chunk is retrieved. Silent overwrite of legal guidance by a newer document is a compliance failure even if the newer document is the correct one — the transition must be governed.

**Decision:**
A conflict detection step runs as a required gate in the reindexing DAG. Before a new regulatory document is ingested, its chunks are compared against the existing index using semantic similarity. Any chunk that contradicts or significantly overlaps with existing guidance is flagged. The new document enters a "pending/under review" state in the index — it is not live for query answering — until a compliance officer reviews and approves the ingestion. Approval is logged to the audit trail. The conflict detection step uses a semantic similarity threshold that is tunable by the compliance team.

**Consequences:**
- Contradictory guidance cannot enter the live index silently; every regulatory update requires a human sign-off on the transition.
- The compliance officer has visibility into what existing guidance is being superseded before approving — this is a material improvement over the current state (PDF searches against a folder of circulars with no version control).
- The reindexing DAG must be maintained and tested; a broken conflict detection step that allows silent ingestion is worse than no conflict detection.
- The semantic similarity threshold for flagging conflicts must be tuned carefully — too low and every document triggers a review; too high and genuine contradictions are missed.
- The "pending/under review" state must be surfaced in a compliance dashboard so that un-reviewed documents do not accumulate and block regulatory updates.

**Alternatives Rejected:**
- *Timestamp-based versioning (newest wins):* rejected because a newer document may not correctly supersede all clauses of the older one; human review is required to confirm the scope of supersession.
- *Parallel indexing with version tagging:* rejected because it pushes the contradiction resolution to query time, where it cannot be governed.

---

## ADR-011: pgvector Over Dedicated Vector Database

**Status:** Proposed

**Context:**
CreditLens requires vector similarity search for RAG retrieval over regulatory documents. A dedicated vector database (Pinecone, Weaviate, Qdrant) would add a separate managed service to the deployment topology. The data volumes in scope (regulatory circulars for a single-country deployment) are in the range of tens of thousands to low hundreds of thousands of chunks — well within the operational range of pgvector running in PostgreSQL.

**Decision:**
Vector embeddings are stored and searched in PostgreSQL using the pgvector extension. No dedicated external vector database is introduced. The regulatory document index lives in the same PostgreSQL cluster as the rest of the application data, with the vector columns in a dedicated schema. HNSW indexing is used for approximate nearest-neighbour search at the expected data scales.

**Consequences:**
- The deployment topology remains PostgreSQL-only for persistence, which is consistent with self-hosted bank infrastructure constraints and reduces the operational surface area.
- Backup, access control, and data sovereignty guarantees are already solved for PostgreSQL — adding pgvector does not require new policies.
- At very large document scales (millions of chunks, sub-100ms latency requirements at high QPS) pgvector may underperform dedicated vector DBs; this is an acceptable trade-off for v1 and early v2 given the expected data volume.
- The team must be trained on pgvector-specific query patterns and HNSW index tuning; this is a smaller knowledge investment than operating a separate vector database.
- If data volume outgrows pgvector's performance envelope, migration to a dedicated vector DB is a contained change (the embedding pipeline and retrieval interface are unchanged; only the storage backend swaps).

**Alternatives Rejected:**
- *Pinecone/Weaviate/Qdrant:* rejected because each adds a managed external service, which conflicts with the self-hosted data sovereignty requirement and the single-bank deployment model.
- *Elasticsearch with vector fields:* rejected because it introduces a second query engine with overlapping but not identical capabilities to PostgreSQL, adding operational complexity without proportionate benefit at this scale.

---

## ADR-012: Database-per-Service Pattern

**Status:** Proposed

**Context:**
CreditLens is composed of multiple services (scoring engine, RAG compliance assistant, audit log, SME portal, override analytics, drift monitoring). Services that share a database schema are coupled at the data layer — a schema change for one service can break another, and a service outage that locks a table can cascade. The self-hosted deployment model means there is no managed database-per-service infrastructure to abstract this; it must be a deliberate architectural choice.

**Decision:**
Each service owns its own database schema, accessible only by that service's application layer. No service reads directly from another service's schema. Cross-service data access is via API or event (RabbitMQ). The audit log service has its own append-only schema that no other service can write to directly — writes go through the audit log service's API. The PostgreSQL deployment may be a single cluster with logical database separation (per-service databases) or separate clusters depending on the bank's infrastructure preference; the isolation contract is enforced at the application layer regardless.

**Consequences:**
- Services can be deployed, upgraded, and scaled independently without cross-service schema coordination.
- The audit log's immutability guarantee is enforced by service boundary — no other service has the database credentials to write to it directly.
- Cross-service queries (e.g., joining a score record with its RAG citations for a report) must be assembled at the application or reporting layer, not via SQL joins — this adds complexity to reporting pipelines.
- The event-driven integration pattern (RabbitMQ) becomes the primary integration mechanism; teams must be aligned on event schema governance and versioning.
- In a single-cluster deployment, database-level access controls (user/role per schema) must be verified in the deployment runbook to enforce the isolation contract.

**Alternatives Rejected:**
- *Shared database, schema-per-service:* rejected because schema-level isolation is weaker than database-level isolation and does not prevent accidental cross-schema queries in application code.
- *Single shared schema:* rejected because it creates tight coupling between services and makes the audit log's immutability guarantee dependent on application-layer enforcement alone.

---

## ADR-013: Post-Disbursement Drift Alerts Routed to Risk Manager, Bypassing Loan Officer

**Status:** Proposed

**Context:**
Post-disbursement monitoring (feature drift on active loans) is structurally ignored in current practice because the loan officer's incentive ends at approval, not at repayment. Routing drift alerts to loan officers produces no action — the officer has moved on. The risk that matters (a cluster of loans approved in the same sector, showing early repayment stress) is a portfolio-level signal, not a single-loan signal, and belongs with the person responsible for portfolio risk.

**Decision:**
Post-disbursement drift alerts (PSI threshold breaches on active loan feature vectors, early repayment stress signals) are routed directly to the risk manager dashboard. They are not routed to the originating loan officer unless the risk manager explicitly escalates a specific case. The drift monitoring DAG reuses the same feature pipeline as scoring; no new infrastructure is required. PSI drift on a portfolio segment is presented to the risk manager as a business signal ("lending into this sector is showing stress, consider a threshold review") not as a DevOps metric.

**Consequences:**
- The risk manager receives actionable portfolio signals before defaults appear in repayment data — this is the primary value of post-disbursement monitoring.
- The loan officer is not burdened with alerts for loans they cannot act on; the routing reflects the actual accountability structure.
- PSI drift becomes a first-class input to the threshold governance process (ADR-006) — the risk manager can initiate a threshold review based on drift data without waiting for defaults.
- The risk manager dashboard must be designed to distinguish "model drift (feature distribution shift)" from "portfolio stress (early repayment indicators)" — they have different remediation paths.
- Individual SME profile monitoring (proactive outreach when a declined SME's profile improves) reuses the same DAG with a different alert target (the assigned loan officer); this is a v2 feature.

**Alternatives Rejected:**
- *Alert all parties (officer + risk manager):* rejected because alert fatigue and diffused accountability produce no action; a single accountable recipient is more effective.
- *Automated threshold adjustment on drift detection:* rejected per ADR-006; drift is an input to a human-governed threshold review, not a trigger for automated policy change.

---

## ADR-014: SME Portal v1 Is Read-Only — Financial Profile Visibility as a Data Rights Implementation

**Status:** Proposed

**Context:**
SMEs currently have no visibility into what data a bank holds about them or how it is used in a credit decision. Under GDPR and applicable BCT rules, they have a right to this information. The SME portal must serve this data rights obligation as a first-class function, not as an afterthought. The Dreamer vision (SME financial passport, portable identity) traces back to this same principle — v1 plants the flag that the SME has a right to see what the system knows about them.

**Decision:**
SME portal v1 is read-only. It exposes: the features extracted from the SME's financial data (with plain-language labels), the score received, the SHAP explanation in plain language, the BCT regulatory articles checked in the RAG layer, and a downloadable PDF financial profile. No write path is exposed in v1. The portal is authenticated via the bank's existing identity provider. The downloadable PDF is a legally meaningful disclosure artifact, not just a convenience export.

**Consequences:**
- Data rights obligations under GDPR Article 15 (right of access) are satisfied structurally — the portal is the access mechanism.
- The SME experiences the system as one that explains itself rather than renders an opaque verdict; this is the "dignity" JTBD from the session.
- The read-only constraint keeps v1 scope manageable; write paths (SME-submitted corrections, document uploads) are deferred to v2 where they require additional data governance design.
- The PDF financial profile becomes the artifact an SME can take to a second bank or a lender — it is the seed of the financial passport concept.
- The portal must not expose raw feature values in ways that enable gaming (e.g., exact cash flow thresholds that trigger scoring tier changes); the SHAP plain-language rendering must be calibrated to explain without enabling threshold gaming.

**Alternatives Rejected:**
- *No SME portal in v1:* rejected because the data rights obligation exists regardless of product roadmap; delaying the portal delays a legal compliance requirement.
- *Full two-way portal (SME can submit corrections) in v1:* rejected because SME-submitted data introduces a new data quality and verification problem that requires separate governance design.

---

*End of ADR starter pack. 14 ADRs. All decisions trace to the brainstorming memlog dated 2026-11-01.*
