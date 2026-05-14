# Prior Authorization — Multi-Agent AI Pipeline (Dynamic Outcomes)

## What This System Does (Plain English)

Imagine a doctor wants to give a patient a special medical device or therapy. The insurance company will not pay unless they pre-approve it. That pre-approval process is called **Prior Authorization (PA)**.

Right now, a billing team manually:
1. Reads through patient PDFs (prescriptions, clinical notes, operative notes)
2. Looks up what the insurance company requires
3. Matches the patient's situation against those requirements
4. Writes out the request and lists the documents to attach

This system does that automatically using a team of specialized AI agents that work together.

```
INPUT:    A folder of patient PDFs
            │
            ▼
SYSTEM:   5 specialized AI agents analyze everything
            │
            ▼
OUTPUT:   "Here is the exact PA argument to submit and
           here is the complete list of documents to attach"
```

The system **only analyzes and produces a recommendation**. A human reviews it and submits manually. The system does not send anything to the insurance company.

---

## Scope

We are building a **Prior Authorization analysis system** that, given clinical data and unstructured PHI delivered as PDFs, uses a multi-agent AI architecture with **dynamic outcomes** to analyze every code, document, and clinical signal and produce:

- The **correct PA recommendation** (what to argue, which codes to cite, which policy criteria are satisfied or unmet)
- The **complete list of required documents** that must accompany that request
---

## 1. AI Models — Why & Where

| Model | Bedrock Model ID | Why this model | Where it is used |
|---|---|---|---|
| **Claude Sonnet 4.6** | `anthropic.claude-sonnet-4-6` | The smartest reasoning brain we have. It can hold up to 1 million words of context in its head at once — that is enough to read every PDF in a case plus the entire insurance policy at the same time. | The Supervisor (the manager agent) and the final synthesis that writes the PA recommendation. |
| **Claude Haiku 4.5** | `anthropic.claude-haiku-4-5` | A faster, cheaper version of Claude — about 5× cheaper. Smart enough for simple jobs like "is this PDF a prescription or a lab report?" | Inside the Document-Ingestion and Coding agents for high-volume, simple-decision tasks. |
| **Amazon Titan Text Embeddings V2** | `amazon.titan-embed-text-v2:0` | Converts text into a list of 1,024 numbers that captures its meaning. This lets us search by *meaning* instead of by exact keyword. Example: searching "back pain" can find policy chunks that say "lumbar discomfort." | Embedding payer policy PDFs once, and embedding each new case query at retrieval time. |
| **Amazon Comprehend Medical** | managed AWS service | A specialized AI trained on millions of medical records. It reads doctor's notes in plain English and automatically maps phrases like "slipped disc at L4-L5" to the standard ICD-10 code `M51.16`. | Inside the Clinical Data Agent. |
| **Amazon Textract** | managed AWS service | An AI that reads scanned/photographed PDFs (where the text is just an image). It can also pull data out of forms — for example, it knows that the value next to the "Patient Name:" label is the patient name. | Inside the Document-Ingestion Agent when a PDF is scanned or has form fields.
---

## 2. Multi-Agents 

**Four reasons to use multi-agent in this project:**

1. **Tool specialization** — Textract is for OCR (Optical Character Recognition), Comprehend Medical is for clinical NLP, Bedrock KB is for policy lookup. Each has different inputs and outputs. Having one prompt try to use all of them is fragile.

2. **Parallelism** — Document ingestion and policy retrieval do not depend on each other, so they can run at the same time. Faster wall-clock time.

3. **Dynamic outcomes** — Each agent returns a partial result. The Supervisor inspects what came back and decides the next move. Example: "Clinical Data Agent found no recent imaging — go back and ask it again with just an imaging filter." The path through the system is decided **per case**, not pre-coded.

4. **Auditability** — Every agent's actions are logged separately. When a HIPAA auditor asks "how did you arrive at this recommendation?", we can show them step-by-step which agent did what and why.
---

## 4. Tools — Why & Where

| Tool | Why it exists (plain English) | Where it is used |
|---|---|---|
| `textract.analyze_document` | Reads scanned PDFs where the text is just a picture. Also pulls form-field values (e.g. the text next to a label). | Document-Ingestion Agent, when a PDF has no text layer or has form fields. | OCR (Optical Character Recognition)
| `pdfplumber.extract_text` | Reads digital PDFs (where the text is already searchable). Free and fast — use this before falling back to Textract. | Document-Ingestion Agent, first attempt before falling back to Textract. |
| `classify_doc_type` | Looks at the extracted text and decides what kind of document it is: prescription, clinical note, lab report, operative note, EOB, ID card, etc. Downstream agents need to know what each document is. | Document-Ingestion Agent. |
| `comprehend_medical.detect_entities_v2` | Finds medical terms in free text (medications, conditions, body parts, lab tests). | Clinical Data Agent. |
| `comprehend_medical.infer_icd10_cm` | Maps free-text diagnoses to ICD-10 codes ("slipped disc" → `M51.16`). | Clinical Data Agent. |
| `comprehend_medical.infer_rx_norm` | Maps medication names to RxNorm codes ("Ibuprofen 400mg" → `5640`). | Clinical Data Agent. |
| `comprehend_medical.detect_phi` | Finds patient-identifying information (names, addresses, SSN) so we can redact it before logging or sending anything outside the AWS BAA boundary. | Every agent, before any logging or external call. |
| `healthlake.create_resource` | Saves the structured clinical data to AWS HealthLake (a HIPAA-safe FHIR database). | Clinical Data Agent — gives us a single source of truth for structured patient data. |
| `kb.retrieve` (Bedrock Knowledge Base) | Searches all payer policy PDFs by *meaning* and returns the 3-5 most relevant chunks. Like a smart Google search over the policy library. | Policy / PA-Criteria Agent. |
| `validate_cpt` / `validate_icd10` / `crosswalk_hcpcs` | Schema-level checks: is this CPT code valid for this ICD-10? Does this HCPCS need a modifier? Are these codes compatible per NCCI edits? | Coding Agent. |
| `lookup_carc_rarc` | Decodes denial codes from the X12 reference list (e   .g. `CO-50` = "non-covered service"). | Coding Agent, when the case includes a prior denial document. |

**Why split tools across agents instead of giving every agent every tool:**
- Shorter system prompts = faster + cheaper
- Less risk of an agent hallucinating a tool name or calling the wrong tool
- Tighter security (IAM): the Document-Ingestion role can call Textract but cannot read the HealthLake database, etc.

---

## 5. The Agents — Why & Where

```
                       ┌────────────────────────────────┐
                       │   SUPERVISOR (Orchestrator)    │
                       │   Claude Sonnet 4.6            │
                       │   Decides next agent to call   │
                       │   Produces final recommendation│
                       └────────────────────────────────┘
                                      │
              ┌──────────────┬────────┴────────┬──────────────┐
              ▼              ▼                 ▼              ▼
       ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ ┌──────────────┐
       │  Document   │ │  Clinical   │ │ Policy / PA  │ │   Coding     │
       │  Ingestion  │ │   Data      │ │  Criteria    │ │   Agent      │
       │  Agent      │ │  Agent      │ │   Agent      │ │              │
       └─────────────┘ └─────────────┘ └──────────────┘ └──────────────┘
```
### 5.1 Document Ingestion Agent

**One-line job:** "Read every uploaded PDF and figure out what each one is."

**Why we need it:**
PHI arrives as PDFs — some are scanned images, some are digital text, some are forms with checkboxes. Nothing downstream can reason about a binary PDF. This agent turns every PDF into clean labeled text plus any form fields and tables that were inside it.

**How it works (step by step):**
1. Receives a new PDF from S3.
2. Tries `pdfplumber.extract_text` first — fast and free for digital PDFs.
3. If that returns empty (scanned PDF), falls back to `textract.analyze_document`.
4. Calls `classify_doc_type` to label what the document is (prescription, op-note, lab, EOB, etc.).
5. Pulls structured form fields where present (every checkbox, every labeled field).

**Where it sits in the flow:**
First step after upload. Runs once per uploaded file. Its output feeds every other agent.

**Tools:** `pdfplumber.extract_text` → fallback to `textract.analyze_document` → `classify_doc_type`.

**Output shape:** `{ doc_id, doc_type, raw_text, form_fields, tables }`.

### 5.2 Clinical Data Agent

**One-line job:** "Turn the doctor's notes into structured medical codes."

**Why we need it:**
Doctors write in human language: *"Patient presents with left shoulder pain, post-surgical, history of arthritis and gout."* Insurance companies only understand codes: ICD-10 for diagnoses, HCPCS/CPT for procedures, RxNorm for medications. Without this translation, the PA recommendation cannot cite anything.

**How it works (step by step):**
1. Receives extracted text from the Document-Ingestion Agent (only for clinical documents).
2. Calls `comprehend_medical.detect_entities_v2` to find every medical term.
3. Calls `infer_icd10_cm` to convert diagnoses into ICD-10 codes.
4. Calls `infer_rx_norm` to convert medication mentions into RxNorm codes.
5. Calls `detect_phi` to redact patient-identifying info before any logging.
6. Builds a FHIR Bundle (a standard healthcare data format) and saves it to AWS HealthLake.

**Where it sits in the flow:**
Runs right after Document Ingestion finishes, but only on documents classified as clinical (notes, labs, imaging, Rx).

**Tools:** `comprehend_medical.detect_entities_v2`, `infer_icd10_cm`, `infer_rx_norm`, `detect_phi`, then `healthlake.create_resource`.

**Output shape:** `{ healthlake_resource_ids, structured_summary: { conditions[], medications[], procedures[], labs[], encounters[] } }`.

### 5.3 Policy / PA-Criteria Agent

**One-line job:** "Find the exact rulebook for this insurer and this procedure."

**Why we need it:**
Every insurance company has different rules. Insurer A might approve a DVT prevention device after any orthopedic surgery; Insurer B might require documented mobility risk factors. The PA recommendation is only valid if it is built against the *current* criteria for *this* insurer for *this* HCPCS code. Hard-coding criteria is not viable — payers update their policies regularly.

**How it works (step by step):**
1. Receives the payer name (already extracted from the LMN/insurance card), the HCPCS code being requested, and the state.
2. Builds a semantic query: *"PA criteria for HCPCS E0676 under [payer] in [state]."*
3. Calls `kb.retrieve` against the Bedrock Knowledge Base, which holds all the payer policy PDFs as searchable chunks.
4. Returns the top 3-5 most relevant chunks **with citations** (which policy PDF, which page).

**Where it sits in the flow:**
Fires as soon as the payer + HCPCS code + state are known (right after Document Ingestion).

**Tools:** `kb.retrieve` against the `policy-kb` Knowledge Base.

**Output shape:** Ordered checklist of criteria, each with the source citation (page-anchored back to the policy PDF), so the final recommendation can cite policy verbatim.

### 5.4 Coding Agent

**One-line job:** "Make sure every code is valid and properly paired."

**Why we need it:**
The single most common reason PA requests fail is bad coding — wrong CPT/ICD-10 pairing, missing modifier, NCCI edit violation, or misreading a CARC/RARC denial code from a prior denial. The PA recommendation has to be built on a clean, validated code set, otherwise the insurer will reject it on technical grounds before even looking at medical necessity.

**How it works (step by step):**
1. Receives the structured clinical summary (from Clinical Data Agent) and the criteria checklist (from Policy Agent).
2. Validates every HCPCS/CPT code with `validate_cpt`.
3. Validates every ICD-10 code with `validate_icd10`.
4. Checks code-pair compatibility and required modifiers via `crosswalk_hcpcs`.
5. If the case included a prior denial EOB, calls `lookup_carc_rarc` to decode the denial reason (e.g. `CO-50` = "non-covered service").

**Where it sits in the flow:**
Runs after Clinical Data Agent and Policy Agent have both produced output.

**Tools:** `validate_cpt`, `validate_icd10`, `crosswalk_hcpcs`, `lookup_carc_rarc`.

**Output shape:** `{ validated_codes[], modifier_recommendations[], coding_issues[], carc_rarc_interpretation }`.

### 5.5 Supervisor (orchestrator)

**One-line job:** "Manage the four specialists and write the final answer."

**Why we need it:**
Someone has to look at all four specialists' outputs together and decide:
- Are any criteria still unmet? (If yes, send a specialist back with a narrower query.)
- Is the data complete enough to build a recommendation? (If yes, stop and synthesize.)
- Which arguments are strong, which are weak?

This is the dynamic-outcomes brain. It is the entry point for every case and the exit point that emits the final structured output.

**How it works (step by step):**
1. Receives the new case (list of uploaded PDFs).
2. Fans out work to specialists in parallel where possible.
3. Reads each specialist's output as it arrives.
4. Detects gaps (e.g. "Criterion 2 needs a recent imaging report — not seen yet").
5. Re-dispatches the relevant specialist with a focused query.
6. Loops until criteria are addressed or marked unattainable.
7. Synthesizes the final PA recommendation + required-documents checklist.

**Where it sits in the flow:**
Entry point + exit point of every case.

**Output (the deliverable of the whole system):**

```json
{
  "case_id": "...",
  "pa_recommendation": {
    "argument": "Patient meets policy criterion 2.1 (post-surgical DVT risk) ...",
    "policy_citations": [{ "policy": "...", "page": 14, "criterion": "2.1" }],
    "supporting_codes": { "hcpcs": ["..."], "icd10": ["..."], "modifiers": ["..."] },
    "confidence": 0.86
  },
  "required_documents": [
    "Completed Prescription / Letter of Medical Necessity",
    "Operative note from surgery",
    "Clinical notes documenting risk factors",
    "Product invoice / MSRP sheet"
  ],
  "open_gaps": [
    "No mobility-assessment note found - request from provider before submission"
  ]
}
```
---

## 6. Data We Need — Required Input Keys

### 6.1 From the Prescription / Letter of Medical Necessity (the most important document)

**Patient Data Keys:**
```
├── patient_name
├── date_of_birth
├── date_of_injury
├── workers_comp_flag        (yes/no)
├── auto_insurance_flag      (yes/no)
├── surgical_patient_flag    (yes/no)
└── surgery_date             (if surgical)
```

**Insurance Data Keys:**
```
├── insurance_carrier        (the payer name)
├── claim_number
├── adjuster_name
└── adjuster_phone
```

**Clinical Data Keys (what is being requested):**
```
├── icd10_codes[]            (diagnosis codes)
├── hcpcs_codes[]            (the devices/therapies requested)
├── body_part                (laterality + region)
├── rental_duration_days     (for rental DME)
└── product_descriptions[]   (free-text descriptions of devices)
```

**Provider Data Keys:**
```
├── provider_name
├── provider_npi
├── provider_signature_present  (yes/no — required for PA)
└── prescription_date
```

### 6.2 From Supporting Clinical Documents

These supplement the Prescription/LMN with medical-necessity evidence:

| Document Type | Keys We Extract |
|---|---|
| Clinical notes (H&P, progress notes) | `risk_factors[]`, `comorbidities[]`, `prior_treatments[]`, `treatment_response` |
| Operative note (if surgical) | `surgery_type`, `surgery_date`, `surgical_findings`, `expected_recovery_period` |
| Lab / imaging report | `test_type`, `test_date`, `findings`, `interpreting_provider` |
| Medication list | `medications[]`, `dosages[]`, `start_dates[]` |
| Prior denial EOB (if any) | `carc_codes[]`, `rarc_codes[]`, `denial_reason_text`, `amount_billed`, `amount_paid` |
| Product invoice / price sheet | `manufacturer`, `part_number`, `msrp`, `acquisition_cost` |

### 6.3 What Happens If a Key is Missing

The Supervisor flags missing keys in the `open_gaps` section of the final output. Example:
- If `surgical_patient_flag` = YES but no operative note is uploaded → flag: *"Operative note required before submission."*
- If `icd10_codes` is empty (left blank on the prescription) → flag: *"ICD-10 code missing on LMN — must be added by provider."*

This is intentional. The system does not invent missing data; it tells the human reviewer exactly what is missing so they can request it before submitting.

---

## 7. Training Data — What Past Data the System Learns From

> Important distinction: we are **not** fine-tuning the LLM itself. Claude and Comprehend Medical are already trained. The "training data" here is what we load into the Knowledge Base and what we use as few-shot examples in agent prompts.

### 7.1 Documents to Load (One-Time + Updates)

| Document Type | What It Provides | Where It Goes |
|---|---|---|
| **Past Prescription / LMN forms** (anonymized) | Examples of what valid PA requests look like. Used as few-shot examples in the Document-Ingestion Agent prompt. | Bundled into agent system prompts (no PHI). |
| **Payer policy PDFs** | The official rules each insurer publishes for each HCPCS/CPT. Updated by payers quarterly. | `s3://policy-kb/` → indexed by Bedrock Knowledge Base. |
| **State fee schedules** | What each state allows providers to charge for each code. | `s3://policy-kb/`. |
| **CARC / RARC code reference list** | Standard denial-code meanings published by X12. | Lookup table in the Coding Agent. |
| **CPT / HCPCS / ICD-10 crosswalks** | Which codes can be paired with which; which need modifiers. | Lookup table in the Coding Agent. |
| **Clinical literature** (peer-reviewed studies) | Used by the Supervisor to strengthen medical-necessity arguments. Example: a study showing DVT risk after orthopedic surgery supports a DVT-prevention-device PA request. | `s3://policy-kb/clinical-literature/`. |
| **Past denial EOBs (835)** with the denial codes the payer used | Helps the Coding Agent learn how each payer interprets each code. | Optional — pattern training only. |

### 7.2 What Each Data Type Trains

| Data Type | What it improves |
|---|---|
| Prescription / LMN examples | Document-Ingestion field-extraction accuracy |
| Payer policy PDFs | Policy/PA-Criteria Agent retrieval relevance |
| State fee schedules | Coding Agent fee-schedule cross-checks |
| CARC / RARC lists | Coding Agent's denial-reason interpretation |
| Clinical literature | Supervisor's argument quality (citable evidence) |
| Past 835 EOBs | Pattern recognition for payer-specific denial habits |

### 7.3 What We Do NOT Need

- **PHI from past cases stored in vector databases.** We do not store patient-identifying data in Pinecone or any external vector store. Patient data lives only in HealthLake (which is HIPAA-safe).
- **Fine-tuned LLM weights.** Claude and Comprehend Medical are already trained; we use them as-is.
- **Past appeal letters** (unless we also have the outcome — approved or denied — otherwise we can't tell if the argument worked).

### 7.4 How We Actually Train the System — Tools & Process

#### The Four Training Destinations

```
┌──────────────────────────────────────────────────────────────────┐
│  1. Bedrock Knowledge Base  ← payer policies + clinical lit       │
│     (vector search by meaning)                                    │
├──────────────────────────────────────────────────────────────────┤
│  2. DynamoDB lookup tables  ← CARC/RARC, CPT/ICD-10 crosswalks    │
│     (exact key-value lookup)                                      │
├──────────────────────────────────────────────────────────────────┤
│  3. S3 reference files      ← state fee schedules (CSV/JSON)      │
│     (loaded into agent memory at boot)                            │
├──────────────────────────────────────────────────────────────────┤
│  4. Agent system prompts    ← few-shot examples of valid LMNs     │
│     (anonymized, embedded directly in the prompt)                 │
└──────────────────────────────────────────────────────────────────┘
```

#### Tools Used For Training

| Tool | Why we use it | What it does |
|---|---|---|
| **AWS CLI** (`aws s3 cp`) | Uploading PDFs and CSVs into S3 buckets | Bulk upload of payer policies and fee schedules |
| **Bedrock Knowledge Base UI** (or `boto3.client("bedrock-agent")`) | Creating + syncing knowledge bases | Auto-chunks PDFs, embeds with Titan V2, builds vector index |
| **`boto3.client("dynamodb")`** | Loading lookup tables | Bulk-writes CARC/RARC/CPT entries as key-value rows |
| **Python ETL scripts** (custom) | Parsing payer-specific CSV/Excel files into clean DynamoDB rows | Standardizes messy source data |
| **`comprehend_medical` validation script** | Sanity-checking that medical text is being parsed correctly | Run a sample of past notes, confirm ICD-10 mappings look right |
| **HealthLake import** (`boto3.client("healthlake").start_fhir_import_job`) | Loading sample FHIR Bundles for testing | Verifies the Clinical Data Agent's write path works |

#### Step-by-Step Process
**Step 1 — Load payer policy PDFs into the Bedrock Knowledge Base**
**Step 2 — Build the CARC/RARC + Code Crosswalk Lookup Tables**
**Step 3 — Load State Fee Schedules**
**Step 5 — Seed HealthLake with sample FHIR Bundles**
**Step 6 — Validate end-to-end with a real (anonymized) case**

Run one full case through the pipeline. Verify in CloudWatch logs that every agent fired, every tool returned data, and the Supervisor's recommendation matches what a human reviewer would have written.

#### Setup-vs-Ongoing Effort

| Step | First-time setup | Ongoing |
|---|---|---|
| 1. Payer policies → Bedrock KB | 1-2 hours | When payer updates (~quarterly) |
| 2. CARC/RARC/CPT lookup tables | 2-4 hours | Annual refresh from X12 |
| 3. State fee schedules | 1 hour per state | When state law changes |
| 4. Few-shot LMN examples | 2 hours | When you add a new doc type |
| 5. HealthLake test bundles | 30 min | One-time |
| 6. End-to-end validation | 1-2 hours | Re-run after any major change |

**Total first-time setup: ~1-2 working days.** Ongoing maintenance: ~1 hour per quarter per payer.

---

## 8. Services — Why & Where
### 8.1 AWS

| Service | Why we use it (plain English) | Where it lives in our system |
|---|---|---|
| **Amazon Bedrock** | Hosts Claude and Titan inside the HIPAA-eligible AWS boundary. Lets us use these models without sending data outside AWS. | All LLM and embedding calls. |
| **Amazon Bedrock AgentCore** | The production runtime for Strands agents. Handles agent memory, identity, secure entry, and full reasoning traces. | Hosts every agent in production. |
| **Amazon Bedrock Knowledge Bases** | A managed Retrieval-Augmented-Generation (RAG) system over PDFs. Splits documents into chunks, embeds them, and serves search-by-meaning. We do not have to build a vector pipeline. | Powers the Policy / PA-Criteria Agent. |
| **AWS HealthLake** | A HIPAA-safe FHIR R4 database — the medical industry's standard format for storing patient data. Single source of truth for structured clinical data. | Output destination of the Clinical Data Agent. |
| **Amazon Comprehend Medical** | The clinical NLP service. Detects medical entities and maps them to standard codes. | Inside the Clinical Data Agent. |
| **Amazon Textract** | The OCR service. Reads scanned PDFs and extracts form fields. | Inside the Document-Ingestion Agent. |
| **Amazon S3** | Encrypted, versioned, tamper-proof storage for the raw PDFs. Object Lock provides WORM (write-once-read-many) retention required for HIPAA. | `phi-pdf-landing/` (raw), `phi-pdf-processed/` (post-ingest), `policy-kb/` (payer policies). |
| **AWS Lambda** | Serverless compute — one Lambda per agent. Each Lambda has its own IAM role with the minimum permissions it needs. Scales automatically. | Compute layer for every agent. |
| **Amazon API Gateway** | The public HTTPS front door for our Next.js dashboard. | Routes upload/read requests to the FastAPI backend. |
| **Amazon EventBridge** | An event router. When a new PDF lands in S3, EventBridge automatically triggers the Supervisor — no polling required. | Triggers the pipeline when a new PDF is uploaded. |
| **AWS KMS** | Manages customer-managed encryption keys. PHI is encrypted at rest with our own keys (not AWS-default keys). | Encryption for S3 buckets and HealthLake. |
| **AWS CloudTrail + CloudWatch** | CloudTrail records every API call (audit log). CloudWatch collects logs and metrics. Together they give us HIPAA-required visibility. | Cross-cutting observability. |
| **AWS Secrets Manager** | Stores any third-party API credentials securely, with automatic rotation. | Read by agents that need external credentials. |
| **Amazon Cognito** | User authentication for the reviewer dashboard. | Front-end auth. |


## 9. Prerequisites — Why & Where

**Beginner intro:**
These are the things that must be set up *before* the system can run. Some are one-time (AWS account, BAA), some are loaded once and updated occasionally (policy PDFs), and some are recurring (each new patient case).

### 9.1 AWS Account & Compliance

| Prerequisite | Why it is required | Where it is set up |
|---|---|---|
| Signed **AWS BAA** (Business Associate Addendum) | PHI cannot legally be processed in an AWS account without an active BAA. This is the HIPAA contract between us and AWS. | AWS Artifact, before any PHI touches the account. |
| **Bedrock model access** for `anthropic.claude-sonnet-4-6`, `anthropic.claude-haiku-4-5`, `amazon.titan-embed-text-v2:0` | Bedrock model access is request-gated per region — you have to ask for it. | Bedrock console, in a HIPAA-eligible region (`us-east-1` or `us-west-2`). |
| **HealthLake datastore** (FHIR R4) | Destination for structured clinical data. | HealthLake console, same region as Bedrock. |
| **S3 buckets** with KMS-CMK encryption, versioning, and Object Lock | PHI must be encrypted at rest with our own keys and protected from deletion (HIPAA WORM). | `phi-pdf-landing/`, `phi-pdf-processed/`, `policy-kb/`. |
| **Bedrock Knowledge Base** over `s3://policy-kb/` | Powers the Policy/PA-Criteria Agent's retrieval. | Bedrock console; ingest payer policy PDFs once and on each update. |
| **Per-agent IAM roles** | Least-privilege scoping — one agent cannot access another's data store. | IAM, one role per Lambda. |

### 9.2 Data the Client Provides (Per Case)

| Input Document | Why it is needed |
|---|---|
| Prescription / Letter of Medical Necessity (PDF) | The primary PA document — contains patient, insurance, clinical, and provider keys. |
| Clinical notes / H&P / progress notes (PDF, often scanned) | Source of medical-necessity evidence beyond what is on the LMN. |
| Operative note (if surgical patient) | Required when surgery is the basis for the request. |
| Lab / imaging reports (if relevant) | Objective evidence supporting policy criteria. |
| Prior denial EOB (if any) | Tells us the exact denial reason to address. |

### 9.3 Reference Data Loaded Once

| Reference Set | Why it is needed | Where it is stored |
|---|---|---|
| Payer policy PDFs | The rulebook for each insurer. Updated on each payer update. | `s3://policy-kb/` → Bedrock KB. |
| State fee schedules | Pricing reference for the Coding Agent. | `s3://policy-kb/`. |
| CARC list — https://x12.org/codes/claim-adjustment-reason-codes | Interpreting payer denial reasons. | Coding Agent lookup table. |
| RARC list — https://x12.org/codes/remittance-advice-remark-codes | Supplementary denial detail. | Same. |
| CPT / HCPCS Level II / ICD-10-CM crosswalks | Code validation. | Same. |
| Clinical literature | Strengthens medical-necessity arguments. | `s3://policy-kb/clinical-literature/`. |

### 9.4 Local Dev Toolchain

```bash
python >= 3.11
pip install strands-agents boto3 fastapi uvicorn \
            pdfplumber PyMuPDF pydantic httpx tenacity
node >= 20      # for the Next.js dashboard
aws-cli v2      # `aws configure sso` against the BAA account
```

---

## 10. Flow of the Project
### 10.1 High-Level Phases

```
        ┌──────────────────────┐
        │  PHASE A — INTAKE     │  Client uploads PHI PDFs
        └──────────┬───────────┘
                   ▼
        ┌──────────────────────┐
        │  PHASE B — UNDERSTAND │  PDFs → text → structured FHIR + codes
        └──────────┬───────────┘
                   ▼
        ┌──────────────────────┐
        │  PHASE C — ANALYZE    │  Match clinical evidence to payer criteria
        └──────────┬───────────┘
                   ▼
        ┌──────────────────────┐
        │  PHASE D — RECOMMEND  │  Output PA recommendation + required docs
        └──────────────────────┘
```

### 10.2 Detailed End-to-End Sequence

```
[1] Client uploads PHI PDFs through the Next.js dashboard.
       └─► API Gateway → FastAPI → S3 (phi-pdf-landing/)   [KMS-encrypted]

[2] S3:ObjectCreated → EventBridge → Supervisor (Claude Sonnet 4.6).
       Supervisor reads the file manifest and builds an initial plan.

[3] Document-Ingestion Agent runs first (every other agent depends on it):
       pdfplumber → fallback Textract → classify_doc_type
       Extracts every Required Input Key from Section 6.
       Output: per-file { doc_type, text, fields, tables }

[4] Parallel fan-out:
    ├─► Clinical Data Agent
    │     (only on clinical documents: notes, labs, op-notes, Rx)
    │     Comprehend Medical → FHIR Bundle → HealthLake
    │     Output: structured clinical summary
    │
    └─► Policy / PA-Criteria Agent
          Uses payer_name + hcpcs_codes + state from the LMN
          kb.retrieve → ordered criteria checklist with page-anchored citations

[5] Coding Agent runs on (clinical summary + proposed codes).
       Validates HCPCS/CPT/ICD-10/modifier combinations.
       If a prior denial EOB exists, decodes CARC/RARC reasons.

[6] Supervisor evaluates completeness:
       IF any criterion lacks supporting evidence → re-dispatch the
       relevant specialist agent with a narrowed query
       (e.g. "Clinical Data Agent, return only mobility/risk-factor notes").
       Loop until criteria are addressed or marked unmet-and-unattainable.
       This is the dynamic-outcome behavior — the path is decided per case.

[7] Supervisor synthesizes the final output:
       {
         pa_recommendation:  { argument, policy_citations, supporting_codes,
                               confidence },
         required_documents: [ ... ],
         open_gaps:          [ ... ]
       }

[8] Output is written to the case record and displayed in the Next.js
    dashboard for a human reviewer to read. The pipeline ends here.
```

### 10.3 Where the Dynamic Outcomes Show Up

| Decision point | Who decides | Inputs that drive the decision |
|---|---|---|
| Which specialist to invoke next | Supervisor LLM | Current case state, which fields are still empty |
| Whether more clinical data is needed | Supervisor LLM | Policy criteria vs. structured clinical summary |
| Which payer policy chunk is "the" rule | Policy Agent (via KB) | `payer_name`, `hcpcs_code`, `state` |
| How to interpret a CARC/RARC code in context | Coding Agent | Code + the specific clinical scenario |
| Final confidence in the PA recommendation | Supervisor LLM | Coverage of criteria, validated codes, open gaps |

None of these are hard-coded — each is decided per case by an LLM reasoning over the evidence actually present.

### 10.4 Observability & Audit

- Every agent tool call is logged to **CloudWatch Logs** under a `case_id` correlation key.
- **AgentCore traces** capture the Supervisor's reasoning chain — used for HIPAA audits.
- All PHI access is auditable via **CloudTrail data events** on the S3 buckets and the HealthLake datastore.

### 10.5 Failure Modes & Fallbacks

| Failure | Fallback |
|---|---|
| Textract cannot OCR a page | Route case to a manual-review queue with the original PDF attached. |
| Comprehend Medical entity confidence < 0.6 | Flag the entity for human confirmation before it influences the recommendation. |
| Supervisor loops > N times without progress | Stop, emit current state as `open_gaps`, hand to human reviewer. |
| Bedrock throttling | Token-bucket retry via `tenacity`; fall back to a secondary region. |
| Required key missing from the LMN (e.g. ICD-10 left blank) | Surface in `open_gaps`; do not invent the missing value. |
