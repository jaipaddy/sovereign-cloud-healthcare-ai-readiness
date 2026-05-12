# Sovereign / Government Cloud Readiness Checklist for Healthcare AI

> Service availability and regulatory status change frequently. Re-verify primary sources (linked at the end) before any commitment.

**For**: Healthcare tech practitioners (platform, security, ML/AI, SRE) deploying AI-enabled workflows in **AWS GovCloud**, **Azure Government**, or **GCP Assured Workloads** — collectively the *government / sovereign cloud regions* in the US.

**How to use**: Walk it with your team in a 60–90 minute session. Score 🟢 / 🟡 / 🔴 per item. Per-domain totals on the rubric at the end.

**Scoring**:
- 🟢 verified available + meets your authorization level
- 🟡 available with a caveat (preview, limited region, lower auth than needed)
- 🔴 unavailable, blocked, or auth gap → blocker

> **🩺 Real-world**: VA's 2025 AI inventory: 367 use cases. HHS: 447. This is no longer hypothetical — it's a real production wave with real procurement, real audits, and real PHI on the line.

---

## Pre-flight: Do you actually need a government / sovereign cloud region?

Required only if at least one is true:

- [ ] FedRAMP High workload (federal agency or contractor data)
- [ ] DoD IL-4 / IL-5 workload
- [ ] CUI / ITAR data
- [ ] Federal customer or state Medicaid contract contractually mandates it
- [ ] Classified data (separate environments)

**HIPAA alone does not require a government / sovereign region.** All three hyperscalers offer HIPAA BAAs in their commercial regions. If your only driver is HIPAA, you're paying a 20–30% premium for compliance you don't need.

---

## Domain 1 — Service Availability

### 1.1 The foundation models you need are available in-region
- **Why**: Model parity lags commercial regions by 6–12+ months. Building on a model not in the government / sovereign region means rewriting later.
- **Verify on AWS**: AWS GovCloud Bedrock doc. FedRAMP High + DoD IL4/5: Claude Sonnet 4.5, Claude 3.7 Sonnet, Claude 3.5 Sonnet v1, Claude 3 Haiku, Llama 3 8B/70B, all Titan models.
- **Verify on Azure**: Azure Government Foundry Models page. `usgovvirginia` / `usgovarizona`: GPT-5.1 (Data Zone Standard), GPT-4.1, GPT-4.1-mini, GPT-4o, o3-mini.
- **Verify on GCP**: Assured Workloads supported services. Vertex AI / Gemini at FedRAMP High; Claude on Vertex AI at FedRAMP High + IL2.
- **Ready looks like**: Your exact model + version is listed today, not "coming soon."

> **🩺 Real-world**: The **FDA's Elsa assistant** (June 2025) runs Claude on Bedrock in AWS GovCloud — used for protocol reviews, adverse-event summarization, label comparisons, and inspection-target prioritization. The same Bedrock+GovCloud stack is generally available to healthcare ISVs.

### 1.2 Healthcare-specific AI services are in-region
- **Why**: NLP, FHIR, and clinical-voice services are common building blocks; their availability gaps trip teams.
- **Verify on AWS**: Comprehend Medical is **GovCloud US-West only**. **HealthLake is NOT in GovCloud** (commercial regions only).
- **Verify on Azure**: Text Analytics for Health, Health Bot, DICOM Service availability in `usgovvirginia` / `usgovarizona`.
- **Verify on GCP**: Cloud Healthcare API availability in your Assured Workloads folder + compliance regime.
- **Ready looks like**: Your specific NLP / FHIR / imaging service is listed in the region's services-in-scope page.

### 1.3 Foundational compute / data services support your runtimes
- **Why**: Government-region Lambda runtimes, EKS/AKS/GKE versions, RDS / Azure SQL / Cloud SQL engine versions, OpenSearch / AI Search / Vertex Vector Search versions trail commercial. CI/CD or IaC may target a version that isn't there.
- **Verify**: Per-service "available regions" pages on each provider.
- **Ready looks like**: Runtime, K8s, and DB versions all GA in your target region.

### 1.4 Third-party AI/ISV dependencies are FedRAMP-authorized
- **Why**: Most healthcare AI ISVs (vector DBs, eval tools, MLOps, scribes) are FedRAMP Moderate or unauthorized, not High.
- **Verify**: FedRAMP Marketplace (marketplace.fedramp.gov) — search vendor + impact level + authorization date.
- **Ready looks like**: Every third-party in your AI stack appears at the impact level you need on the cloud you chose.

### 1.5 Decision: do you need multi-region DR at all?
- **Why**: Multi-region in a sovereign cloud roughly **doubles infrastructure cost** and significantly increases architectural complexity. Many healthcare AI workflows (research, internal scribing, batch analytics, prior-auth assist) tolerate hours of downtime; some (active clinical decision support during patient encounters, ED triage) do not. Decide before you architect, not after.
- **Verify**: Walk the workflow with clinical / business stakeholders and answer:
  - What does the workflow do during a regional outage — fall back to manual, queue, or fail?
  - What's the patient-safety impact of a 4-hour outage? A 24-hour outage?
  - What's the contractual / regulatory uptime commitment (BAA, customer SLA, state Medicaid contract)?
  - Is the data reproducible (rebuild from EHR) or only-here (model fine-tunes, prompt logs, audit trail)?
- **Ready looks like**: Documented decision — *single-region* or *multi-region* — signed off by clinical leadership and the business owner, with the workflow's tier (see 1.5a / 1.5b).

### 1.5a If single-region — what RPO / RTO can you actually commit to?
- **Why**: Single-region in a sovereign cloud is a real choice, but it changes the SLA you can promise. **Single-region does not mean single-AZ** — in AWS GovCloud (and analogously Azure Government / GCP Assured Workloads), production deployments must be **multi-AZ within the region**. The provider's HIPAA/FedRAMP-aligned reference architectures assume multi-AZ; running single-AZ in production for PHI workloads is a finding waiting to happen. Multi-AZ buys you AZ-level resilience for free (or close to it); the gap that remains is the *regional* outage.
- **Typical achievable targets — multi-AZ within a single sovereign region** (with disciplined backups + IaC):
  - **RPO**: 15 min – 1 hour (driven by snapshot / continuous-backup cadence: RDS PITR, Azure SQL PITR, Cloud SQL PITR — all give ~5-min granularity within region).
  - **RTO for AZ-level failure**: **seconds to minutes**, automatic via the managed service's multi-AZ failover (RDS Multi-AZ, ElastiCache, Aurora, Azure SQL zone-redundant, Cloud SQL HA). No human intervention.
  - **RTO for regional control-plane / service degradation (region still up)**: 1 – 4 hours, often via redeploy into healthy AZs in the same region.
  - **RTO for full regional outage**: measured in days. You cannot fail over — you wait, or you accept the outage and rebuild from cross-region backups *if* you've been replicating them.
- **Verify**:
  - **Multi-AZ is on by default for every stateful service**: RDS / Aurora Multi-AZ, ElastiCache Multi-AZ, OpenSearch multi-AZ with standby, S3 (inherently multi-AZ), EFS regional, MSK multi-AZ — and the Azure / GCP equivalents (zone-redundant SKUs, HA configs).
  - Compute tier spans ≥ 2 AZs behind a regional load balancer; ASG/VMSS/MIG min count ≥ AZ count.
  - K8s control plane is regional / multi-AZ (EKS / AKS / GKE regional clusters), worker nodes spread across AZs.
  - AI-specific stateful items are multi-AZ: vector stores (OpenSearch / AI Search / Vertex Vector Search), prompt-and-response logs, model artifacts, Bedrock Guardrails / Content Safety / Model Armor configurations.
  - Backups are stored **cross-AZ within the region** by default; consider also replicating to the *other* sovereign region of the partition for regional-outage protection (this is the pragmatic middle path between true single-region and full multi-region).
  - PITR enabled on every stateful service.
  - IaC (Terraform / CloudFormation / Bicep / Deployment Manager) can rebuild the workload from zero, tested.
  - Documented "regional outage" runbook that explicitly says *we wait* (or *we restore from cross-region backup, accepting hours of RTO*) and the business has accepted that.
- **Ready looks like**: Multi-AZ posture verified across every stateful and stateless tier. RPO/RTO numbers split into three columns — *AZ failure*, *regional degradation*, *regional outage* — written down, signed off, and reflected in the customer-facing SLA. Backup restore drills run quarterly; AZ-failover behavior tested at least annually (chaos / fault injection).

### 1.5b If multi-region — what do RPO / RTO actually require?
- **Why**: Multi-region across the partition is the only way to survive a regional outage. The pattern (active/active vs. active/passive vs. pilot light) determines whether you hit minutes-RTO or hours-RTO.
- **Region-pair availability** (your pattern is bounded by geography):
  - **AWS GovCloud**: 2 regions — `us-gov-west-1` ↔ `us-gov-east-1`, **3 AZs each**. That's your pair.
  - **Azure Government**: `usgov-virginia`, `usgov-arizona`, `usgov-iowa`, `usgov-texas` (plus DoD-only `us-dod-central`, `us-dod-east`). Confirm AI-service availability matches across both regions of your chosen pair — gaps exist (e.g., GPT-5.1 Provisioned managed is currently uneven across `usgov-arizona` / `usgov-virginia`).
  - **GCP Assured Workloads**: 9 US regions; pick a pair within your Assured Workloads folder + same compliance regime.
- **Pattern → typical RPO / RTO**:
  | Pattern | RPO | RTO | Cost multiplier | When to use |
  |---|---|---|---|---|
  | Backup & restore (single-region with cross-region backups) | 1–4 hours | 4–24 hours | ~1.1x | Tier 3 (back-office, batch) |
  | Pilot light | 5–15 min | 30 min – 2 hours | ~1.3x | Tier 2 (clinical-adjacent) |
  | Warm standby (active/passive) | < 5 min | 5–30 min | ~1.6x | Tier 1 (clinical decision support) |
  | Active/active (multi-region writes) | Near zero | < 5 min | ~2x | Tier 0 (real-time patient safety) |
- **Verify**:
  - Every stateful service supports **cross-region replication within the sovereign partition** — do not rely on a commercial-region pattern that doesn't exist in sovereign (e.g., some services have cross-region in commercial but not in GovCloud).
  - **AI-specific** stateful items are covered: vector stores (OpenSearch / AI Search / Vertex Vector Search), prompt-and-response logs, fine-tuned model artifacts, model evaluation datasets, Bedrock Guardrails / Content Safety / Model Armor configurations.
  - Failed-over workflow keeps the same model + version available in the secondary region (parity gap is the silent killer here).
  - Federated identity (IdP) and KMS keys are replicated or accessible from the secondary region; key custody is documented.
  - Full failover **and failback** drill executed end-to-end at least annually, with PHI-realistic data volume.
- **Ready looks like**: Tier classification documented per workflow; RPO/RTO targets per tier; pattern chosen per tier; last drill report dated within 12 months and showed actual RTO ≤ target.

### 1.5c Cross-region AI parity check (often missed)
- **Why**: Even when the *region pair exists*, the *AI services + model versions* available in each region may differ. Failover to a region where your model isn't available is not failover.
- **Verify**: For each model / AI service in your workflow, confirm it is GA in **both** regions of your pair, at the **same version**, with the **same authorization level**.
- **Ready looks like**: A two-column table (primary region, secondary region) with each AI dependency, both green.

---

## Domain 2 — Observability

### 2.1 Native logs / metrics / traces have parity with commercial
- **Why**: CloudWatch Logs Insights, Azure Monitor Logs, Cloud Logging — feature flags differ between commercial and sovereign.
- **Verify**: Provider-specific "feature differences in [GovCloud/Azure Government/Assured Workloads]" docs.
- **Ready looks like**: Every dashboard, query, and alert from your commercial-region observability stack works in sovereign.

### 2.2 LLM-specific observability is wired up
- **Why**: Token-level metering, prompt/response logging, guardrail traces, model evaluation results — not all are GA in sovereign regions.
- **Verify on AWS**: Bedrock invocation logging to S3/CloudWatch, Bedrock Guardrails traces, Bedrock model evaluation availability in GovCloud.
- **Verify on Azure**: Azure OpenAI request/response logging via diagnostic settings; AI Content Safety logs; Azure AI Foundry observability features per region.
- **Verify on GCP**: Vertex AI logging via Cloud Logging, Model Armor / Safety filters audit trails.
- **Ready looks like**: Every LLM call is logged, every guardrail decision is auditable, retention meets HIPAA accounting-of-disclosures.

### 2.3 Third-party APM / SIEM is available at your impact level
- **Why**: Observability vendors authorize unevenly across government clouds.
  - **Datadog for Government**: **FedRAMP High Authorized as of May 6, 2026** (FedRAMP Moderate previously authorized). Runs in a physically isolated gov instance — commercial-tenant dashboards, integrations, and data do not transfer; provisioning is separate.
  - **Splunk Cloud**: FedRAMP Moderate / DoD IL2. **Splunk Observability Cloud** is FedRAMP Moderate *in process* on AWS only — not offered on Azure or GCP.
  - **Elastic Cloud Hosted**: FedRAMP High on AWS GovCloud.
  - **New Relic**: FedRAMP Moderate.
- **Verify**: FedRAMP Marketplace records dated within 12 months.
- **Ready looks like**: Your SIEM/APM is authorized at the level you need, *on the cloud you chose*.

### 2.4 Audit / config / change tracking is complete
- **Verify on AWS**: CloudTrail (data events on Bedrock/SageMaker/HealthLake), AWS Config, CloudTrail Lake.
- **Verify on Azure**: Activity Log, Diagnostic Settings, Defender for Cloud, Microsoft Sentinel.
- **Verify on GCP**: Cloud Audit Logs (Admin / Data Access / System Event), Asset Inventory, Security Command Center.
- **Ready looks like**: Every PHI access and AI inference is captured with WORM-grade retention.

---

## Domain 3 — Security

### 3.1 Encryption + key management at FIPS 140-2 (3 where required)
- **Verify on AWS**: KMS in GovCloud (FIPS 140-2 endpoints), CloudHSM availability.
- **Verify on Azure**: Azure Key Vault Managed HSM (FIPS 140-2 Level 3) availability per region.
- **Verify on GCP**: Cloud KMS + Cloud HSM in Assured Workloads.
- **Ready looks like**: BYOK / HYOK path documented; FIPS endpoints used everywhere.

### 3.2 IdP federation works for your identity stack
- **Why**: Entity IDs and SAML/OIDC endpoints differ in sovereign regions; conditional access, MFA, and JIT provisioning can break silently.
- **Verify**: Test SAML / OIDC federation against sovereign endpoints (different ARN partition / different tenant URLs).
- **Ready looks like**: SSO + MFA + break-glass tested end-to-end against sovereign endpoints.

### 3.3 Native threat detection is in-region and at your impact level
- **AWS**: GuardDuty, Security Hub, Inspector, Macie in GovCloud (verify each).
- **Azure**: Defender for Cloud + Defender for various workloads (Azure Government).
- **GCP**: Security Command Center Premium / Enterprise (Assured Workloads coverage).
- **Ready looks like**: All four pillars (threat detection, vuln scanning, posture, data classification) covered.

### 3.4 Network controls only egress to sovereign endpoints
- **Why**: A misconfigured PrivateLink / Private Endpoint / PSC can silently call commercial endpoints, breaking data residency.
- **Verify**: Egress logs show zero traffic to commercial-partition service endpoints. Confirm CloudFormation / ARM / Terraform providers use sovereign-partition ARNs/URIs.
- **Ready looks like**: Network egress is whitelisted to sovereign endpoints only; commercial endpoints are denied at SCP / Azure Policy / Org Policy.

### 3.5 BAA scope covers every service in your AI workflow
- **Why**: Provider BAA covers a list of services. Anything outside is on you.
- **Verify on AWS**: AWS HIPAA Eligible Services list — check every service in your workflow.
- **Verify on Azure**: Microsoft Trust Center HIPAA-covered services. **Azure OpenAI Realtime API (audio in/out) is NOT yet HIPAA-covered as of preview** — do not send PHI through it.
- **Verify on GCP**: Google Cloud HIPAA-covered services list.
- **Ready looks like**: Every service in your architecture is on the BAA list, with an effective signed BAA.

### 3.6 Confidential computing path is available if required
- **AWS**: Nitro Enclaves availability in GovCloud per instance family.
- **Azure**: Confidential VMs / Confidential Containers in Azure Government.
- **GCP**: Confidential VMs / Confidential GKE Nodes in Assured Workloads.
- **Ready looks like**: If you need attestation or memory encryption, the SKU exists in the sovereign region.

---

## Domain 4 — AI Risk Management

### 4.1 NIST AI RMF mapping exists for the workflow
- **Why**: NIST AI RMF 1.0 (Govern / Map / Measure / Manage) is the de facto framework regulators and customers reference.
- **Verify**: Documented AI use case → risk register → controls mapping per the four functions.
- **Ready looks like**: One-page RMF map per AI workflow, owned by a named risk owner.

### 4.2 HHS OCR + Section 1557 obligations are met
- **Why**:
  - **Section 1557 final rule** (effective Jul 5, 2024) — its **AI / Patient Care Decision Support Tools (PCDSTs) provision** has a delayed compliance date of **May 1, 2025**. Covered entities must make reasonable efforts to identify PCDSTs that use input variables accounting for race, color, national origin, sex, age, or disability — and mitigate discrimination risk.
  - **Partial non-enforcement (May 13, 2025)**: HHS announced it will not enforce portions of § 92.210(b)–(c) tied to **gender identity / pregnancy / abortion** sex-discrimination claims while litigation proceeds. **The race, color, national origin, age, and disability components remain enforceable** — and those are exactly what most healthcare-AI bias work targets. Do not read the notice as a free pass on PCDST obligations.
  - **HHS OCR HIPAA Security Rule NPRM** (proposed Dec 27, 2024; published in Federal Register Jan 6, 2025) — **still proposed; not yet finalized as of 2026-05-12**. Finalization remains on OCR's regulatory agenda for May 2026 but has not published. It treats AI tools as in-scope of risk analysis and asset inventory. Treat as directional, not yet binding, but design as if it will be.
- **Verify**: Risk analysis lists every AI tool that creates/receives/maintains/transmits ePHI. Bias testing artifacts exist for any clinical-decision-supporting model. PCDST inventory is current.
- **Ready looks like**: Documented risk analysis + bias testing tied to specific model versions; PCDST inventory dated within 6 months.

### 4.3 Native guardrails / safety filters are configured
- **AWS**: Bedrock Guardrails (PII detection, denied topics, content filters, contextual grounding) — confirmed available with Claude in AWS GovCloud.
- **Azure**: Azure AI Content Safety configured on every Azure OpenAI deployment.
- **GCP**: Vertex AI Safety filters + Model Armor configured on every Gemini / Claude endpoint.
- **Ready looks like**: Guardrails enabled by default; bypass requires named approval; failures are logged.

### 4.4 Human-in-the-loop and clinical override paths are documented
- **Why**: Clinical AI without an explicit override path triggers regulatory and malpractice exposure. CA SB 1120 ("Physicians Make Decisions Act," effective Jan 1, 2025) explicitly requires human oversight for AI in health-insurance utilization management.
- **Verify**: Workflow diagrams show where a clinician confirms or overrides; logs capture overrides; SB 1120 applies if you operate in California utilization management.
- **Ready looks like**: Override path tested in production-like environment; override rate monitored.

> **🩺 Real-world**: **VA GPT** has 95,000+ employees onboarded for admin tasks (drafting, summarization). VAEC built on AWS + Azure (both FedRAMP High). This is the scale you're competing with on workflow polish.

### 4.5 Model evaluation, red-teaming, and bias testing on PHI-scope data
- **Why**: PHI cannot leave the sovereign region for evaluation either — eval pipelines must run in-region.
- **Verify**: Eval datasets, eval runs, and red-team artifacts live in sovereign storage.
- **Ready looks like**: Evals are reproducible, dated, and tied to model version. Bias metrics include demographic slices.

### 4.6 Model card / system card / AI incident response runbook
- **Verify**: Model card per model version. Incident runbook covers hallucination, prompt injection, data leakage, and model degradation.
- **Ready looks like**: On-call has the runbook bookmarked; tabletop has been run within 6 months.

### 4.7 ISO/IEC 42001 alignment (if pursuing certification)
- **Why**: Increasingly expected by enterprise health-system buyers.
- **Verify**: Gap assessment against ISO/IEC 42001:2023 controls.
- **Ready looks like**: Either a certification roadmap exists, or a documented decision not to pursue.

---

## Domain 5 — FinOps

### 5.1 Sovereign-region price premium is modeled
- **Why**: AWS GovCloud runs ~20–30% over commercial (e.g., m5.large is ~26% higher in us-gov-east-1). Azure Government and GCP Assured Workloads carry similar premiums.
- **Verify**: Pull last 90 days of commercial-region usage and re-price against sovereign-region SKUs in the provider pricing calculator.
- **Ready looks like**: TCO model has the premium baked in, with explicit line items per service.

### 5.2 AI token / inference pricing is modeled in sovereign
- **Why**: Bedrock token pricing in GovCloud, Azure OpenAI in Azure Government, and Vertex AI in Assured Workloads each price differently from commercial.
- **Verify**: Provider pricing docs for sovereign region; estimate at projected token volume.
- **Ready looks like**: Per-workflow inference cost estimate, with sensitivity to ±50% token usage.

### 5.3 Native cost tooling has parity
- **AWS**: Cost Explorer, Budgets, Cost and Usage Report (CUR) in GovCloud — confirm CUR delivery to GovCloud S3 bucket.
- **Azure**: Microsoft Cost Management in Azure Government.
- **GCP**: Cloud Billing + BigQuery export in Assured Workloads.
- **Ready looks like**: Same dashboards, same anomaly detection, same alerts as commercial.

### 5.4 Commit-discount instruments cover sovereign SKUs
- **Verify**: Savings Plans / RIs (AWS), Reservations / Savings Plans (Azure), CUDs (GCP) — all apply only within their partition. Coverage planning must be sovereign-specific.
- **Ready looks like**: Commit coverage target (e.g., 70%) defined for sovereign workloads independently from commercial.

### 5.5 Cross-partition billing constraint understood
- **Why**: Sovereign accounts generally cannot consolidate billing with commercial accounts. AWS GovCloud accounts must link to a paired commercial account but bill separately.
- **Verify**: Org structure and payer-account mapping documented.
- **Ready looks like**: Finance has a single source of truth across partitions and a defined showback model.

### 5.6 Egress pricing modeled (RAG-killer)
- **Why**: Egress out of sovereign regions is often more expensive than commercial; cross-partition egress for RAG over commercial-region data sources is a budget trap.
- **Verify**: Architecture diagrams show no egress for steady-state RAG.
- **Ready looks like**: All retrieval / vector store / data sources for any AI workflow live within the sovereign partition.

### 5.7 Tag / label governance per AI workflow
- **Why**: Per-model and per-workflow chargeback is impossible without disciplined tagging on inference endpoints, vector stores, and storage.
- **Verify**: Tag policy enforced via SCP / Azure Policy / Organization Policy. Untagged spend < 5% of total.
- **Ready looks like**: Monthly chargeback report by AI workflow lands automatically.

> **🩺 Real-world**: **Covered California** uses **Vertex AI / Gemini** in Assured Workloads to verify ~50,000 healthcare documents per month at 84% accuracy. Token-and-doc cost is the main FinOps lever; tagging by workflow is what made chargeback defensible.

---

## Domain 6 — Workforce & Personnel

> Mostly relevant for IL4 / IL5 / ITAR. Skip if your only driver is FedRAMP High.

### 6.1 US persons rule met
- **Why**: For IL4/IL5/ITAR workloads, all CSP and customer support personnel with access to data must be **US citizens, US nationals, or US persons (green-card holders)**. This applies to your support staff and anyone with privileged access — including third-party SI partners.
- **Verify**: Provider support routing is in a US-persons-only queue (e.g., Microsoft 365 GCC High routes only screened US persons). Your own access list reviewed; contractor agreements include the US-persons clause.
- **Ready looks like**: Quarterly attestation that every privileged identity is a US person.

### 6.2 Background screening + clearances
- **Why**: Healthcare + government work imposes baseline screening above standard HR; IL5 / classified work requires clearances.
- **Verify**: NACI/T1 (or equivalent) for federal-data handlers; clearances for IL5; healthcare-grade background check for PHI access.
- **Ready looks like**: Roster of cleared personnel by data class, refreshed annually.

### 6.3 Privileged access governance
- **Why**: Standing root / global admin is a finding under HIPAA, FedRAMP, and emerging HHS OCR rules.
- **Verify**: JIT/PIM with auditable approval; break-glass paths logged and alerted; AI-engineering identities cannot directly query PHI tables; separation of duties between model training and PHI access.
- **Ready looks like**: Zero standing-privilege accounts; break-glass usage reviewed monthly.

### 6.4 Training & insider-risk monitoring
- **Why**: AI introduces novel insider-risk patterns: mass prompt extraction, model-weight exfiltration, training-data tampering.
- **Verify**: Annual HIPAA + AI-specific training (NIST AI RMF awareness); behavioral monitoring includes AI-specific detections (high-volume PHI-prompt extraction, unusual model-artifact access).
- **Ready looks like**: Training attestation + a tested AI-insider-risk detection.

---

## Domain 7 — Data Lifecycle & Privacy

> Goes beyond HIPAA — state laws and AI-specific data flows that practitioners often miss.

### 7.1 Data residency & flow mapping
- **Why**: A misconfigured pipeline can silently call commercial-region endpoints. Data sovereignty is broken at that point.
- **Verify**: Every PHI flow mapped; egress to commercial-partition endpoints denied at SCP / Azure Policy / Org Policy. No cross-partition replication.
- **Ready looks like**: Flow diagram dated within 6 months; egress logs show zero commercial-endpoint traffic.

### 7.2 Retention & deletion policy
- **Why**: PHI, prompts, completions, embeddings, fine-tune training sets, and eval data all have different retention drivers (HIPAA, state law, BAA, customer contracts).
- **Verify**: Policy per data class; "right to delete" tested end-to-end (including from vector stores and prompt logs); deletion is verifiable, not just a soft flag.
- **Ready looks like**: Quarterly proof of deletion test from a dedicated test record.

### 7.3 Minimum necessary applied to prompts
- **Why**: Sending entire patient records to an LLM violates minimum-necessary and inflates leakage blast radius.
- **Verify**: Pipeline redaction / tokenization / structured-extraction step before prompts. Prompt logs reviewed for over-disclosure.
- **Ready looks like**: Prompt schema enforces minimum-necessary fields per use case.

### 7.4 De-identification path documented
- **Why**: De-identified data unlocks training / eval without HIPAA constraints — but only if done correctly.
- **Verify**: HIPAA Safe Harbor or Expert Determination method documented per dataset; expert determinations dated and attributed.
- **Ready looks like**: De-id method, dataset, and date are linked to every training / eval artifact.

### 7.5 State privacy laws inventory
- **Why**: Several state laws cover *health data* outside HIPAA scope and apply even to non-covered entities. Most carry private right of action or significant per-violation penalties.
- **Verify**: Compliance posture documented for the laws that apply to you:
  - **WA My Health My Data Act** — broad scope (non-HIPAA health data), opt-in consent, private right of action. Effective Mar 31, 2024.
  - **CA CMIA + CPRA** — sensitive PI now includes health and neural data.
  - **TX HB 300** — stricter than HIPAA for entities operating in Texas.
  - **CA SB 1120** — human oversight for AI in health-insurance utilization management. Effective Jan 1, 2025.
  - **NY HIPA** — pending; track status.
- **Ready looks like**: One-page state-law applicability matrix per AI workflow.

### 7.6 Patient consent / disclosure
- **Why**: Several state laws (and sector-specific rules) require disclosure when AI is used in coverage or care decisions.
- **Verify**: Disclosure language vetted by counsel; consent UX tested where required.
- **Ready looks like**: Patient-facing disclosure exists where the workflow is consumer-facing.

### 7.7 Model training boundary enforced
- **Why**: All three hyperscalers contract-default that customer prompts and outputs are not used to train foundation models — but the technical control must still be in place.
- **Verify**: AWS Bedrock — confirm no opt-in to model improvement; Azure OpenAI — verify Data, privacy, and security defaults; GCP Vertex AI — verify "your data is yours" terms in your contract.
- **Ready looks like**: Both contractual and technical guarantees documented; auditor can trace one prompt's lifecycle and confirm it never left your tenant.

---

## Scoring Rubric

| Domain | Items | Total |
|---|---|---|
| 1. Service Availability | 1.1–1.4, 1.5, 1.5a *or* 1.5b, 1.5c | __ / 7 |
| 2. Observability | 2.1–2.4 | __ / 4 |
| 3. Security | 3.1–3.6 | __ / 6 |
| 4. AI Risk Management | 4.1–4.7 | __ / 7 |
| 5. FinOps | 5.1–5.7 | __ / 7 |
| 6. Workforce & Personnel | 6.1–6.4 | __ / 4 |
| 7. Data Lifecycle & Privacy | 7.1–7.7 | __ / 7 |
| **Overall** | | **__ / 42** |

**Bands**:
- **36–42 🟢** — Ready to migrate / launch.
- **27–35 🟡** — Conditional go; close gaps before production PHI.
- **<27 🔴** — Not ready. Run a 60-day remediation sprint before commitments.

**Stop-ship rules** (override the total):
- Any 🔴 in **Service Availability** or **Security** → stop-ship.
- Any 🔴 in **6.1 (US persons)** if your workload is IL4/IL5/ITAR → stop-ship.
- Any 🔴 in **7.1 (data residency)** → stop-ship.

---

## Regulatory Status (as of 2026-05-12)

Tag every regulation with current status before citing it in customer or auditor conversations. This list ages — re-verify before commitments.

| Item | Status | Key dates |
|---|---|---|
| **HIPAA Security Rule overhaul** | **Proposed** (NPRM) — not yet finalized | Published Federal Register Jan 6, 2025. Comments closed Mar 7, 2025. Finalization remains on OCR's regulatory agenda for May 2026 but has not yet published as of 2026-05-12. |
| **Section 1557 final rule** | **Final** (with partial non-enforcement) | Effective Jul 5, 2024. AI / PCDST provision compliance: **May 1, 2025**. **HHS issued partial non-enforcement on May 13, 2025** for § 92.210(b)–(c) gender-identity / pregnancy / abortion claims; race / color / national origin / age / disability components remain enforceable. |
| **NIST AI RMF 1.0** | **Published** (voluntary) | Released Jan 2023. Generative AI Profile (NIST AI 600-1) Jul 2024. |
| **ISO/IEC 42001:2023** | **Published** (voluntary cert.) | Released Dec 2023. |
| **FDA AI/ML SaMD lifecycle guidance** | **Draft** | Published Jan 6, 2025 ("AI-Enabled Device Software Functions: Lifecycle Management and Marketing Submission Recommendations"). |
| **EU AI Act** | **Phased** | In force Aug 1, 2024. High-risk AI obligations apply Aug 2, 2026. |
| **Washington My Health My Data Act** | **In force** | Effective Mar 31, 2024 (small biz Jun 30, 2024). Private right of action. |
| **California SB 1120** | **In force** | Effective Jan 1, 2025. AI in utilization management. |
| **FedRAMP 20x program** | **Phase 2 complete; Phase 3 scheduled** | Launched Mar 2025. Phase 2 pilot wrapped Mar 31, 2026 (13 cloud services participated). Phase 3 — wide-scale Low/Moderate adoption — scheduled Q3–Q4 2026. Existing FedRAMP authorizations remain valid. |

---

## Sources

All sources accessed within the last 12 months. Primary sources favored.

**AWS**
- [Amazon Bedrock in AWS GovCloud (US)](https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/govcloud-bedrock.html)
- [Amazon Comprehend Medical in AWS GovCloud (US)](https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/govcloud-cmpm.html)
- [Bedrock model support by Region](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html)
- [Claude in Amazon Bedrock — FedRAMP High and DoD IL4/5 (Anthropic)](https://www.anthropic.com/news/claude-in-amazon-bedrock-fedramp-high)
- [Bedrock models get FedRAMP High and DoD IL-4/5 in GovCloud (AWS What's New, May 2025)](https://aws.amazon.com/about-aws/whats-new/2025/05/amazon-bedrock-models-fedramp-high-dod-il-4-5-govcloud/)
- [Using AWS GovCloud (US) Regions — 3 AZs per region](https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/using-govcloud.html)

**Azure**
- [Foundry Models sold directly by Azure in Azure Government](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-gov)
- [Azure OpenAI in Azure Government](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/azure-government)
- [Azure Government regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)
- [Azure OpenAI Realtime API HIPAA eligibility (Microsoft Q&A)](https://learn.microsoft.com/en-us/answers/questions/5616040/clarification-request-hipaa-eligibility-of-azure-o)

**GCP**
- [Vertex AI Search and Generative AI achieve FedRAMP High](https://cloud.google.com/blog/topics/public-sector/vertex-ai-search-and-generative-ai-with-gemini-achieve-fedramp-high)
- [More FedRAMP High services in Assured Workloads](https://cloud.google.com/blog/products/identity-security/more-fedramp-high-authorized-services-are-now-available-in-assured-workloads)
- [Claude on Vertex AI — FedRAMP High and IL2](https://claude.com/blog/claude-on-google-cloud-fedramp-high)
- [Deployment guidance for Gemini for Government](https://docs.cloud.google.com/architecture/security/deploy-gemini-gov)
- [Assured Workloads](https://cloud.google.com/security/products/assured-workloads)

**Regulations & frameworks**
- [HIPAA Security Rule NPRM — Federal Register, Jan 6, 2025](https://www.federalregister.gov/documents/2025/01/06/2024-30983/hipaa-security-rule-to-strengthen-the-cybersecurity-of-electronic-protected-health-information)
- [Section 1557 Final Rule — Federal Register, May 6, 2024](https://www.federalregister.gov/documents/2024/05/06/2024-08711/nondiscrimination-in-health-programs-and-activities)
- [Section 1557 partial non-enforcement notice — May 13, 2025 (gender-identity / pregnancy / abortion claims)](https://baldwin.com/compliance/aca-section-1557-rulemaking-notice-of-non-enforcement/)
- [FedRAMP 20x Phase 2 cohort & Phase 3 timeline](https://www.executivegov.com/articles/fedramp-20x-phase-2-cohort1-2026-plans)
- [NIST AI Risk Management Framework 1.0](https://www.nist.gov/itl/ai-risk-management-framework)
- [FDA Artificial Intelligence in SaMD](https://www.fda.gov/medical-devices/software-medical-device-samd/artificial-intelligence-software-medical-device)
- [Washington My Health My Data Act — IAPP overview](https://iapp.org/resources/article/washington-my-health-my-data-act-overview)
- [FedRAMP Marketplace](https://marketplace.fedramp.gov/)
- [FedRAMP 20x program update](https://www.fedramp.gov/2025-03-24-FedRAMP-in-2025/)

**Third-party SaaS**
- [Elastic Cloud Hosted FedRAMP High on AWS GovCloud](https://ir.elastic.co/news/news-details/2026/Elastic-Cloud-Hosted-Achieves-FedRAMP-High-Authorization/default.aspx)
- [Datadog for Government — FedRAMP High Authorized (May 6, 2026)](https://www.datadoghq.com/about/latest-news/press-releases/datadog-for-government-achieves-fedramp-high-certification/)
- [Splunk Observability Cloud FedRAMP support](https://help.splunk.com/en/splunk-observability-cloud/fedramp-support)

**Real-world references**
- [Anthropic — Claude in Bedrock: FedRAMP High & DoD IL4/5 (FDA Elsa context)](https://www.anthropic.com/news/claude-in-amazon-bedrock-fedramp-high)
- [VA Artificial Intelligence — Use Case Inventory + AI strategy](https://department.va.gov/ai/)
- [Bipartisan Policy Center — Federal Health Agency AI](https://bipartisanpolicy.org/article/mapping-the-rise-of-ai-in-federal-health-agencies/)

**FinOps**
- [GovCloud Pricing 2026 (CapLinked)](https://www.caplinked.com/blog/govcloud-pricing-in-2026-understanding-costs-and-maximizing-roi/)
- [Azure Government vs AWS GovCloud (VSO)](https://vso-inc.com/azure-government-cloud-vs-aws-govcloud-a-2026-cost-and-capability-comparison-for-defense-contractors/)
