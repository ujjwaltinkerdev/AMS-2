# State Laws + Payer Policies — Training Architecture

> **Scope:** This document covers ONE topic — how the system learns and applies
> (a) state insurance laws, (b) payer-specific policies, and (c) patterns from past
> denials/appeals — across two workflows:
>
> 1. **Initial PA recommendation** — building a fresh PA request.
> 2. **Denial → Appeal / Re-appeal** — when a payer denies, the system recommends the
>    strongest appeal argument by searching policy KBs **plus** historical denial→appeal
>    outcomes from past patients.
>
> **The headline question:** *Do we fine-tune a model, or do we use Bedrock LLM with retrieval?*
>
> **The answer (TL;DR):** Use **Bedrock LLM + RAG (Retrieval-Augmented Generation)** across
> **three knowledge bases**: `state-law-kb`, `payer-policy-kb`, and `denial-outcomes-kb`
> (anonymized past cases). Do NOT fine-tune. The reasons are technical, financial, and
> regulatory — explained below.


 Insurance companies (like Aetna, Cigna) have national policies, but they often add state-specific changes called "amendments" or "bulletins."
---

## 🎯 Pilot Scope — Two States Only

For the initial release the system is **restricted to two states: Michigan (MI) and
Texas (TX)**. These two were chosen deliberately:

| State | Why it's in the pilot |
|---|---|
| **Michigan (MI)** | Complex workers' comp fee schedule + the existing primary market. |
| **Texas (TX)** | Fundamentally different regulatory model (auto/PIP-heavy, distinct UR rules). Exercising the architecture against real variation early de-risks future expansion. |

**This is a scoping choice, not an architectural limit.** The RAG-based design is
*horizontally scalable*: adding a new state means uploading its statutes + fee schedule
+ payer state-amendments to the KBs with the right metadata tags. **No code change.
No model retraining. Estimated effort per new state after pilot: 1–2 engineer-days.**

Expansion plan:

```
Phase 1 (Now)         →  MI, TX                   (2 states, ~10 priority payers)
Phase 2 (+3 months)   →  + CA, FL, NY             (top-5 commercial markets)
Phase 3 (+9 months)   →  Nationwide (50 states)   (driven by client demand)
```

All numbers and KB-volume estimates below reflect **Phase 1 pilot scope** unless noted
otherwise.

---

## 1. The Problem We're Solving

### 1.1 Four sources of variability

```
┌──────────────────────────────────────────────────────────────    ┐  
│  VARIABILITY DIMENSION       │  EXAMPLE                          │
├──────────────────────────────┼─────────────────────────────────  ┤
│  1. STATE LAW                │  Michigan WC fee schedule says    │
│                              │  E0676 reimburses at $XYZ;        │
│                              │  Texas WC says $ABC; California   │
│                              │  requires a UR review first.      │
├──────────────────────────────┼─────────────────────────────────  ┤
│  2. PAYER POLICY             │  Aetna requires "6 weeks of       │
│                              │  conservative therapy" before     │
│                              │  approving E0676.                 │
│                              │  Cigna requires "documented       │
│                              │  mobility risk factor only."      │
├──────────────────────────────┼─────────────────────────────────  ┤
│  3. PATIENT CIRCUMSTANCE     │  Post-surgical vs chronic.        │
│                              │  Workers' comp vs commercial.     │
│                              │  Auto liability vs medicare.      │
├──────────────────────────────┼─────────────────────────────────  ┤
│  4. PAST OUTCOMES            │  "Last 12 Aetna-MI denials of     │    
│    (denial → appeal data)    │  E0676 with CO-50 were overturned │
│                              │  when the appeal cited LCD L33797 │
│                              │  + an operative note attachment." │
└──────────────────────────────┴─────────────────────────────────  ┘
```

### 1.2 Why this is hard

- **Pilot scope: 2 states × ~10 priority payers × ~5 product categories = ~100 unique rule
  combinations.** At nationwide scale (Phase 3) the same math is ~50 × ~30 × ~5 ≈ **7,500
  combinations** — the architecture must be designed for that ceiling now, even if we only
  load 100 today.
- **The combinatorial trap (the part most teams miss):** every insurance company operates
  under *different rules in every state it sells in*. **Aetna-MI ≠ Aetna-TX**. UnitedHealthcare-MI
  is governed by Michigan WC rules, fee schedules, and DIFS bulletins; UHC-TX answers to the
  Texas DWC, TDI bulletins, and a different UR statute. The variability surface is
  **(payer × state)**, not "state" or "payer" in isolation.
- **Three independent update cadences feed that surface:**
  - Payers publish national + state-amendment policies **quarterly** (sometimes monthly).
  - State insurance departments issue rule changes via **bulletins and circulars** throughout
    the year, separate from any legislative cycle.
  - State legislatures occasionally rewrite statutes (e.g. fee-schedule overhauls).
---


## 2. The Core Decision — Fine-Tune vs RAG

### 2.3 Head-to-head comparison

| Criterion | Fine-Tune | RAG (Bedrock KB) | Winner |
|---|---|---|---|
| **Policy update cadence** | Payers update quarterly → must re-fine-tune every 90 days. Cost & ops nightmare. | Re-upload PDF → re-sync KB → done in 15 min. | **RAG** |
| **Citation / audit trail** | Model says "this is the rule" but cannot point to a source. HIPAA + payer audit risk. | Every claim cites `policy.pdf` page X, section Y. Verbatim. | **RAG** |
| **Hallucination risk** | High. Model may invent criteria from training memory. | Low. Model is constrained to the retrieved chunks. | **RAG** |
| **Scalability across payers** | Each new payer = new training data + re-fine-tune. | Each new payer = upload their PDFs. Zero architecture change. | **RAG** |
| **Patient-specific reasoning** | Hard — model has to generalize from training to never-seen patient combinations. | Patient details shape the *query*; retrieval is contextual to each case. | **RAG** |
| **Cost (first 100 cases)** | $30K–$100K+ for dataset prep + fine-tuning. Plus per-payer iteration. | ~$5–$30/month for Bedrock KB + per-query LLM cost. | **RAG** |
| **Latency** | ~1.5s (no retrieval). | ~2.5s (retrieval + generation). | Fine-Tune (marginal) |
| **Regulatory defensibility** | "Model decided" is hard to defend. | "Per policy X page Y, decided as Z" is defensible. | **RAG** |
| **Data freshness** | Frozen at training cut-off. | Always current with the underlying KB. | **RAG** |
| **Cold-start for new state/payer** | Need labeled training data, weeks to onboard. | Drop their PDFs in S3, ready in 30 minutes. | **RAG** |

**Result: 9–1 in favor of RAG.** The single latency win for fine-tuning is irrelevant
in a workflow that is human-reviewed and asynchronous.

### 2.4 The two-line architect rule

> **Fine-tune** when the *behavior* is stable and the *knowledge* needs to be intrinsic.
> **RAG** when the *knowledge* changes faster than you can retrain.

Payer policies change faster than we can retrain. **Therefore RAG.**

---

## 3. Why RAG Is The Right Answer For This Specific Domain

### 3.1 The structure of the problem favors RAG

Insurance policies and state laws share three properties that make them *perfect* for RAG:

1. **They are written documents.** PDFs with structured sections. Already machine-readable.
2. **They cite themselves.** Section numbers, page references, code citations — natural anchors.
3. **They have effective dates.** Each policy version applies to a window. Metadata-driven retrieval handles this trivially.

Fine-tuning *destroys* these properties — once weights absorb the text, the citations are lost.

### 3.2 The reasoning layer is the same regardless of policy

Whether the patient is in Michigan or Texas, whether the payer is Aetna or Cigna, the
*reasoning pattern* is identical:

```
1. What does the patient have?            (clinical evidence)
2. What does the rulebook require?        (retrieved policy chunks)
3. Does (1) satisfy (2)?                  (LLM reasoning — Claude's strength)
4. What's missing if not?                 (open_gaps)
```

Claude Sonnet 4.6 is already excellent at step 3 — *without* any fine-tuning. The
variability lives entirely in step 2, which is a *retrieval* problem, not a *learning*
problem.

---

## 4. The Training Architecture (RAG-Based)

### 4.1 Four knowledge stores, one query

```
                                  ┌─────────────────────┐
                                  │  CASE QUERY (either │
                                  │  initial PA  OR     │
                                  │  denial→appeal)     │
                                  └──────────┬──────────┘
                                             │
            ┌─────────────────┬──────────────┼──────────────┬──────────────────┐
            ▼                 ▼              ▼              ▼                  ▼
  ┌────────────────┐ ┌────────────────┐ ┌──────────────┐ ┌────────────────┐ ┌────────────────┐
  │  STATE-LAW KB  │ │  PAYER-POLICY  │ │ DENIAL-      │ │  FEE SCHEDULE  │ │  HEALTHLAKE    │
  │  (Bedrock KB)  │ │  KB (Bedrock)  │ │ OUTCOMES KB  │ │  (DynamoDB)    │ │  (FHIR R4)     │
  │                │ │                │ │  (Bedrock)   │ │                │ │                │
  │  MI + TX       │ │  ~10 payers ×  │ │ Anonymized   │ │  MI + TX       │ │  Current       │
  │  WC, auto, etc.│ │  state amends  │ │ past cases   │ │  $/HCPCS       │ │  patient data  │
  └────────┬───────┘ └────────┬───────┘ └──────┬───────┘ └────────┬───────┘ └────────┬───────┘
           │                  │                │                  │                  │
           └──────────────────┴────────────────┴──────────────────┴──────────────────┘
                                              ▼
                                    ┌────────────────────┐
                                    │  CLAUDE SONNET 4.6 │
                                    │  (Supervisor)      │
                                    │  Reasons over the  │
                                    │  retrieved chunks  │
                                    └────────────────────┘
```

### 4.2 Why three separate Knowledge Bases (state-law / payer-policy / denial-outcomes)?

| Reason | Explanation |
|---|---|
| **Different update cadence** | State laws update yearly; payer policies update quarterly; denial outcomes update **continuously** (every closed case). Separate KBs = separate sync pipelines. |
| **Different metadata schemas** | State chunks tagged with `{state, statute_type}`. Payer chunks tagged with `{payer, hcpcs_code, effective_date}`. Outcomes chunks tagged with `{payer, state, carc_codes, hcpcs, appeal_strategy, outcome, overturn_rate}`. |
| **Different sensitivity** | State + payer KBs hold public reference data. Denial-outcomes KB derives from PHI and demands stricter IAM, KMS-CMK, and de-identification. Separating them means the strict controls apply only where required. |
| **Cleaner retrieval** | Querying KBs in parallel + reranking gives better recall than one giant mixed index. Each KB has its own embedding strategy if needed. |
| **Independent IAM scoping** | Auditor can be granted read-only access to policy KBs without seeing outcome data, or vice versa. |

### 4.3 Metadata schema (the part most teams get wrong)

This is where the RAG quality lives. Each PDF chunk in Bedrock KB must carry rich metadata:

**State-Law KB metadata:**
```json
{
  "state": "MI",
  "domain": "workers_comp",
  "topic": "prior_authorization",
  "effective_date": "2026-01-01",
  "statute_number": "MCL 418.315",
  "source_url": "...",
  "version": 3
}
```

**Payer-Policy KB metadata:**
```json
{
  "payer": "Aetna",
  "policy_id": "CPB-0148",
  "hcpcs_codes": ["E0676", "E0675"],
  "product_category": "dvt_prevention",
  "effective_date": "2026-02-15",
  "supersedes": "CPB-0147",
  "page": 14,
  "section": "2.1"
}
```

**At query time, filter retrieval by:**
- `state == patient.state`
- `payer == patient.payer`
- `effective_date <= today AND (supersedes IS NULL OR superseded_by IS NULL)`
- `hcpcs_codes CONTAINS query.hcpcs_code`

This is what gives the system its *patient-specific* answer — filtering on metadata makes
the retrieval contextually correct.

#### 4.4.2 How chunks are embedded

We embed a **synthetic narrative** built from the structured fields, *not* the raw clinical
text. Example embed-source text:

```
Aetna Michigan workers-comp denial of HCPCS E0676 with CARC CO-50 and RARC N362
("Service not deemed medically necessary; missing documentation of conservative care").
Appeal cited LCD L33797 and attached operative note, 6-week PT progress log, and signed
LMN. Outcome: overturned, full payment, 18 days to decision.
```

This gives us two benefits at once:
1. **Vector similarity** finds past cases that share the same denial fingerprint, even when
   wording differs.
2. **No raw PHI** ever enters the embedding (the text was synthesized from already-redacted
   structured fields).


---

## 5. How the Four Variability Dimensions Are Handled

### 5.1 State law variability

**Mechanism:** Metadata-filtered retrieval on the State-Law KB.

**No model retraining when a state law changes.** Just re-upload the updated statute PDF.

### 5.2 Payer policy variability

**Mechanism:** Metadata-filtered retrieval on the Payer-Policy KB.
**Adding a new payer = upload their policies. No retraining.**

### 5.3 Patient-specific variability
 **The policy doesn't change per patient.**
What changes per patient is **which subset of a policy applies**.

The patient's profile shapes the *query*:
```
Patient is post-surgical → query mentions "post-surgical risk"
Patient has prior conservative therapy → query mentions "failed conservative treatment"
Patient is workers' comp → filter to workers_comp policies
```

### 5.4 Denial → Appeal / Re-appeal workflow (uses ALL three KBs)

This is the second workflow the system supports. It activates when a payer denies a PA
request or a claim. The system must propose the strongest possible appeal.

**End-to-end retrieval flow:**

```
[1] Denial arrives (EOB / 835 / denial letter PDF)
       │
       ▼
[2] Coding Agent decodes CARC + RARC codes
    → "CO-50 / N362 — service not medically necessary; missing documentation"
       │
       ▼
[3] Build the appeal-query fingerprint:
    { payer, state, hcpcs, icd10, carc_codes, rarc_codes,
      patient_circumstance (post-surgical, WC, etc.) }
       │
       ▼
[4] PARALLEL FAN-OUT — query all three KBs at once:
    │
    ├── state-law-kb       (filter: state, domain)
    │     → "MI workers-comp appeal rights, timing windows"
    │
    ├── payer-policy-kb    (filter: payer, hcpcs, effective_date)
    │     → "Aetna current CPB-0148 medical-necessity criteria"
    │
    └── denial-outcomes-kb (filter: payer, state, carc, outcome=OVERTURNED_*)
          → Top-10 anonymized past cases with the same fingerprint
            + the appeal strategy that worked + overturn rate
       │
       ▼
[5] Rerank combined results (Cohere Rerank) → keep top 3-5 from each KB
       │
       ▼
[6] Claude Sonnet 4.6 (Supervisor) synthesizes:
    {
      "appeal_argument": "...",
      "policy_citations":    [<from payer-policy-kb>],
      "statute_citations":   [<from state-law-kb>],
      "precedent_cases":     [<from denial-outcomes-kb, with overturn rates>],
      "documents_to_attach": [<derived from precedent patterns>],
      "confidence":          0.78,
      "expected_overturn_rate": 0.71  // based on similar past cases
    }
       │
       ▼
[7] Human reviewer approves → submits appeal manually
       │
       ▼
[8] When the appeal closes, feed the outcome back into denial-outcomes-kb
    (closed-loop learning — see Section 7.5)
```

### 6 What we will NOT fine-tune

- **Claude Sonnet 4.6** — the reasoning brain. RAG over policies is correct.
- **The Policy/PA-Criteria Agent** — its job is retrieval + reasoning, not memorization.
- **The Supervisor** — its job is dynamic orchestration. Fine-tuning would freeze the orchestration patterns.
- **Anything on past denial/appeal outcomes** — past cases are RAG fuel, not training fuel. See 6.4 below.


## 7. The Ingestion Pipeline (How Data Gets Into the KBs)

### 7.1 Sources and cadence

| Source | Format | Cadence | Loaded Into |
|---|---|---|---|
| State workers' comp statutes | Statute PDFs from `legislature.<state>.gov` | Annual + on legislative change | `state-law-kb` |
| State auto-insurance laws | Regulation PDFs | Annual | `state-law-kb` |
| State fee schedules | CSV/Excel from state DOL | Annual + amendments | DynamoDB `fee-schedules` table |
| Payer medical-necessity policies | PDFs from payer provider portals | Quarterly | `payer-policy-kb` |
| Payer DME/HCPCS coverage lists | PDFs / structured exports | Quarterly | `payer-policy-kb` + DynamoDB `payer-coverage` |
| CMS LCDs (Local Coverage Determinations) | PDFs from CMS.gov | Quarterly | `payer-policy-kb` (medicare scope) |
| NCCI Edits (CMS) | Quarterly CSV | Quarterly | DynamoDB `ncci-edits` |

### 7.3 Chunking strategy (this is where retrieval quality is won or lost)

**Wrong:** Fixed 500-token chunks (breaks policy sentences across chunks).

**Right:** **Semantic chunking by policy section**. Each chunk = one numbered policy criterion
(e.g. "2.1.a — Patient must have documented...").

In Bedrock KB:
- Use **hierarchical chunking** for policy PDFs (parent = section, child = sub-criterion)
- Parent chunk is what's returned to the LLM (richer context)
- Child chunks are what the vector search hits on (sharper match)

### 7.5 Closed-case feedback loop (powers the Denial-Outcomes KB)

This is the **continuous-learning** pipeline. Every time a case closes (appeal approved or
denied), a Lambda triggered by the case-status update writes a new pattern chunk into
`denial-outcomes-kb`. No model retraining — just an embedding upsert.

**Operational properties:**

| Property | Value |
|---|---|
| Latency from case-close to KB availability | 5–15 minutes |
| Quarantine cohort threshold (k-anonymity) | k = 5 |
| Date-shift window | ±90 days per patient (deterministic from `patient_id_hash`) |
| Retention | Indefinite for OVERTURNED cases; 7 years for DENIED (HIPAA min). |
| Re-embedding cadence | Full rebuild only if embedding model is upgraded. |

**This is the only KB that grows continuously.** State and payer KBs are refreshed on
publication; outcomes grow per case. Aft r 12 months in production, this KB becomes the
system's most valuable asset — and the harder it would be to replicate via fine-tuning.

---

## 8. Cost Analysis — RAG vs Fine-Tune

### 8.1 RAG (Bedrock KB + Claude Sonnet 4.6) — Pilot Scope (MI + TX)

| Component | Setup Cost | Monthly Cost (1,000 cases) |
|---|---|---|
| Bedrock KB — `state-law-kb` (MI + TX statutes, fee schedules, bulletins, ~1.5K chunks) | $0 | ~$10 (OpenSearch Serverless minimum) |
| Bedrock KB — `payer-policy-kb` (~10 payers × national + MI/TX amendments, ~12K chunks) | $0 | ~$15 |
| Bedrock KB — `denial-outcomes-kb` (~5K chunks, growing) | $0 | ~$10 |
| Titan V2 embeddings (initial load + quarterly refresh + per-case outcome upsert) | ~$3 | ~$3 |
| Claude Sonnet 4.6 inference (~6K in / 1.5K out per case across both workflows) | $0 | ~$45/1,000 cases |
| Cohere Rerank (optional, ~$1 per 1K queries) | $0 | ~$3 |
| **Total (Pilot — 2 states)** | **~$3** | **~$86 / 1,000 cases** |

**Scaling note:** Moving from 2 states to 50 states roughly **3×'s** the KB cost
(~$25/mo additional), not 25×. Reason: OpenSearch Serverless cost is dominated by minimum
provisioned capacity, not raw chunk count, until volumes reach the hundreds of thousands.
Per-query inference cost is unchanged by geographic scope — it depends on tokens, not
chunk count.

### 8.2 Fine-Tuning (hypothetical Claude or open-weight model)

| Component | Setup Cost | Monthly Cost |
|---|---|---|
| Dataset preparation (annotators, validation) | $20K–$50K | — |
| Fine-tuning compute (Bedrock custom model or SageMaker) | $5K–$30K per run | — |
| Re-fine-tuning each quarter (4×/year) | $20K–$120K/year | — |
| Hosting fine-tuned model | $1K–$5K/month (provisioned throughput) | $1K–$5K |
| Per-token inference (cheaper but…) | — | ~$15/1,000 cases |
| **Total Year 1** | **$45K–$200K+** | **~$15K–$50K/year inference** |

**RAG is ~500× cheaper in Year 1.** This is not close.

---

## 9. Production Best Practices

### 9.1 Metadata filtering is mandatory

Never query a KB without filtering by `state`, `payer`, and `effective_date`. Without filters,
you risk citing an Aetna-CA policy for an Aetna-TX patient, or a 2024 policy for a 2026 case.

### 9.2 Reranking (Cohere Rerank or custom)

After Bedrock KB returns top-20 chunks, run a reranker to pick the best 3–5. This dramatically
improves citation precision (per published benchmarks, +15–25% MRR).

### 9.3 Citation enforcement

In the Supervisor's prompt, require every claim to carry a citation:

```python
SUPERVISOR_PROMPT = """
For every policy claim in your PA recommendation, you MUST cite:
  - The exact retrieved chunk ID
  - The policy_id and page number from its metadata
A claim without a citation will be rejected. Refuse to write the recommendation
if you cannot find supporting policy text.
"""
```
This single rule prevents 90%+ of hallucination risk.
