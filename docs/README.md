# Prior Authorization — Multi-Agent AI Pipeline

> **8 seconds. 1 LMN. Full PA recommendation with citations.**

A doctor submits a **Letter of Medical Necessity (LMN)** — a structured PDF form with pre-printed ICD-10/HCPCS codes and checkboxes. Our pipeline reads the form, looks up the payer's PA criteria, validates the codes, and produces a PA recommendation + open-gaps checklist. A human reviewer submits the final result.

---

## Quick Summary

| | |
|---|---|
| **Input** | 1 LMN PDF per case |
| **Output** | PA recommendation JSON + open-gaps checklist |
| **Scope** | Analysis only — system never submits to the payer |
| **Cost** | ~$0.60/case at 1K cases/mo → ~$0.29/case at 10K |
| **Compliance** | HIPAA-eligible (AWS BAA + KMS-CMK + WORM S3) |
| **Avg. processing time** | ~8 seconds end-to-end |

---

## The 4 Agents

| Agent | Model | Job |
|---|---|---|
| 🧾 **Document Ingestion** | Claude Haiku 4.5 | Extract every field + checkbox from the LMN PDF |
| 📚 **Policy / PA-Criteria** | Claude Sonnet 4.6 | Find the exact payer rulebook for the HCPCS codes |
| 🔢 **Coding** | Claude Haiku 4.5 | Validate codes + modifiers + NCCI compatibility |
| 🎯 **Supervisor** | Claude Sonnet 4.6 | Orchestrate specialists, detect gaps, write final answer |

---

## Start Here

- [☁️ Complete Services Reference](/PRIOR_AUTHORIZATION_MULTIAGENT_PIPELINE#1-complete-services-reference-top-of-document) — every AWS service, cost, and why
- [🤖 Multi-Agent Architecture](/PRIOR_AUTHORIZATION_MULTIAGENT_PIPELINE#3-multi-agent-architecture) — how the 4 agents work together
- [🔄 End-to-End Flow](/PRIOR_AUTHORIZATION_MULTIAGENT_PIPELINE#4-end-to-end-technical-flow-stage-by-stage) — stage-by-stage walkthrough
- [💰 Cost Summary](/PRIOR_AUTHORIZATION_MULTIAGENT_PIPELINE#5-cost-summary) — pilot → nationwide pricing
- [📝 Sample LMN Input](/PRIOR_AUTHORIZATION_MULTIAGENT_PIPELINE#8-sample-input-script) — Leonardo's real case

---

## Sample Output (Leonardo's Case)

```json
{
  "case_id": "case-12345",
  "pa_recommendation": {
    "argument": "Patient meets Gallagher Bassett WC criterion 2.1 for E1399 — surgical knee, 60-day rental, valid NPI 1225091143.",
    "policy_citations": [
      { "policy": "GB-WC-DME-v3.2", "page": 14, "criterion": "2.1" }
    ],
    "supporting_codes": {
      "hcpcs": ["E1399", "E0221"],
      "icd10": ["M75.82"],
      "modifiers": ["LT", "RR"]
    },
    "confidence": 0.86
  },
  "open_gaps": [
    "Surgical Patient = Y but no operative note uploaded — request from Dr. Bridges"
  ]
}
```
