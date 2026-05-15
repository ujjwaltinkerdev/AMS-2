# Prior Authorization — Multi-Agent AI Pipeline

## What This System Does

A doctor sends a **Letter of Medical Necessity (LMN)** — a structured PDF form with pre-printed ICD-10/HCPCS codes and checkboxes for body part, laterality, rental duration, and surgical status. Our pipeline reads the form, looks up the payer's PA criteria, validates the codes, and produces a Prior Authorization recommendation + a list of any missing documents. A human reviewer submits the final result manually.

**Input:** 1 LMN PDF per case (structured form — see [sample script](#sample-input-script))
**Output:** PA recommendation JSON + open-gaps checklist
**Scope:** Analysis only — system never submits to the payer

---

## 1. Complete Services Reference (Top of Document)

This is the canonical list of every AWS service used, why we picked it, what it costs, and what we'd swap to if we removed it.

| # | Service | Scope | Why We Use It | Example | Pilot Cost (1K cases/mo) | Alternative If Removed |
|---|---|---|---|---|---|---|
| 1 | **Amazon Textract** (Forms + Queries + Tables) | OCR + form-field + checkbox extraction from structured PDFs | The LMN is a structured form. Textract reads labeled fields and detects checked/unchecked boxes natively. | Reads `ICD-10: M75.82`; detects ✅ on `E1399` and `E0221` boxes; reads `Insurance Carrier: Gallagher Bassett` | OCR only ~$0.0015/page, Table ~$0.015/page,Forms	~$0.05/page  | AWS Bedrock Data Automation, Google Document AI, Azure Form Recognizer, manual entry |
| 2 | **Amazon Bedrock — Claude Sonnet 4.6** | Heavy LLM reasoning — Supervisor orchestration + PA recommendation synthesis | 1M-token context fits the entire payer policy + extracted form data in one prompt. Strongest medical reasoning. | Writes: *"Patient meets Gallagher Bassett WC criterion 2.1 for E1399 — surgical knee, 60-day rental, valid NPI 1225091143."* | **~$46.50** ($3 input / $15 output per 1M tokens) | GPT-4 (no HIPAA BAA), self-hosted Llama 70B (ops overhead) |
| 3 | **Amazon Bedrock — Claude Haiku 4.5** | Lightweight LLM tasks — doc classification, field normalization | 5× cheaper than Sonnet for simple yes/no jobs. | Classifies *"this PDF is an LMN"*; normalizes `04/06/2026` → ISO date | **~$0.80** ($0.80 input / $4 output per 1M tokens) | Claude Sonnet (5× cost), open-source small LLMs |
| 4 | **Amazon Titan Text Embeddings V2** | Vector embeddings for semantic search over payer policies | Converts policy text + agent queries into 1024-dim vectors. HIPAA-eligible inside Bedrock. | Embeds *"PA criteria for E1399 under Gallagher Bassett WC in Florida"* → vector | **~$0.01** ($0.00002 / 1K tokens) | Cohere Embed v3, OpenAI text-embedding-3 (no BAA) |
| 5 | **Amazon Bedrock Knowledge Bases** (OpenSearch Serverless backend) | Managed RAG over 5,000+ payer policy PDFs — chunks, embeds, indexes, retrieves with citations | Removes the need to build/maintain a vector pipeline. Returns page-anchored citations for every result. | Returns: *"Gallagher Bassett policy v3.2, page 14, criterion 2.1"* | **~$346.00** (2 OCU minimum × $0.24 × 720h) | Pinecone, Aurora `pgvector`, Weaviate, Qdrant |
| 6 | **Amazon Bedrock AgentCore** | Production runtime for multi-agent systems — memory, IAM identity per agent, secure entry, full reasoning traces | Bundles agent hosting + session memory + audit traces. Without it we'd manually wire Lambda + DynamoDB + CloudWatch. | Supervisor remembers *"Ingestion returned E1399, M75.82"* across multiple tool calls within the same case | **~$60.00** (~$0.015 × 4,000 invocations) | Lambda + DynamoDB session store + custom structured logging |
| 7 | **Amazon S3** (with Object Lock + KMS-CMK) | Encrypted, versioned, WORM-compliant storage for PHI PDFs and payer policies | HIPAA requires WORM retention + encryption at rest with customer-managed keys. | `s3://phi-pdf-landing/case-12345/lmn.pdf` | **~$1.10** ($0.023/GB + minimal requests) | Azure Blob, GCS — but breaks the AWS BAA boundary |
| 8 | **AWS Lambda** | Serverless compute — one Lambda per agent | Auto-scales, least-privilege IAM per agent, no servers to manage | `DocIngestionLambda` triggered when LMN lands in S3 | **~$0.00** (within free tier) | EC2 (idle cost), Fargate (more ops) |
| 9 | **Amazon API Gateway** (REST) | Public HTTPS endpoint for the Next.js reviewer dashboard | TLS termination, throttling, auth integration with Cognito | `POST /upload` → Lambda → S3 | **~$0.01** ($3.50 / 1M requests) | ALB, CloudFront Functions |
| 10 | **Amazon EventBridge** | Event router — triggers Supervisor when a PDF lands in S3 | Eliminates polling; clean event-driven trigger | `S3:ObjectCreated` → `SupervisorLambda` | **~$0.001** ($1 / 1M events) | Direct S3 → Lambda trigger (less flexible) |
| 11 | **AWS KMS** | Customer-managed encryption keys for S3, CloudWatch logs, Secrets Manager | HIPAA requires CMKs (not AWS-default keys) | Encrypts every PHI PDF at rest with our own key | **~$3.15** ($1/key/mo + API calls) | AWS-managed keys (not HIPAA-eligible for PHI), CloudHSM (overkill) |
| 12 | **Amazon CloudWatch** | Centralized logs + metrics + alarms | Per-agent structured logs correlated by `case_id` | Logs: *"DocIngestionAgent took 2.3s for case-12345"* | **~$5.50** ($0.50/GB ingestion) | Datadog, New Relic (extra cost + egress) |
| 13 | **AWS CloudTrail** | Audit trail of every AWS API call (incl. S3 data events) | HIPAA-required audit log of who accessed which PHI when | Logs: *"reviewer@example.com read lmn.pdf at 2026-05-15T10:23Z"* | **~$0.01** ($0.10 / 100K data events) | Manual logging (insufficient for HIPAA) |
| 14 | **AWS Secrets Manager** | Stores third-party credentials with auto-rotation | KMS-encrypted, IAM-scoped, automatic rotation | Stores payer-portal API tokens | **~$1.20** ($0.40/secret/mo) | SSM Parameter Store (no rotation), env vars (insecure) |

---

## 2. Detailed Service Breakdown — What Each Service Does (Simple Language)

This section walks through every service from the table above, in plain English, with a real example from Leonardo's case and the exact per-task cost.

---

### 2.1 Amazon Textract — "The PDF Reader"

**What it does (simple):**
Reads a PDF and pulls out text, form fields, and checkboxes. Think of it as a smart scanner that understands forms.

**Example with Leonardo's LMN:**
```
Leonardo's LMN PDF lands in S3
        ↓
Textract reads page 1
        ↓
Returns structured data:
   "Patient Name" → "Leonardo Mesa Verdecia"
   "ICD-10 Code"  → "M75.82"
   "E1399 box"    → ✅ CHECKED
   "E0221 box"    → ✅ CHECKED
   "Laterality"   → "Left" (checkbox detected)
   "Rental"       → "60 Days" (checkbox detected)
```

**Detailed Costing:**

| Feature | Price | What It Does |
|---------|-------|--------------|
| **OCR only (text)** | $0.0015/page | Reads plain text from PDF |
| **Forms feature** | $0.05/page | Reads labeled fields (Name: ___) |
| **Tables feature** | $0.015/page | Reads tabular data |
| **Queries feature** | $0.015/page | Custom questions like "What is the NPI?" |

**Per Case Cost for LMN (1 page, all features):**
```
1 page × $0.05 (Forms)   = $0.05
1 page × $0.015 (Tables) = $0.015
1 page × $0.015 (Queries)= $0.015
                          --------
                  Total: $0.08 per case
```

**Monthly cost (1,000 cases × $0.08) = $80/month**

---

### 2.2 Amazon Bedrock — Claude Sonnet 4.6 — "The Smart Thinker"

**What it does (simple):**
The brain of the system. Used by Supervisor and Policy Agent for complex reasoning — reading policies, making decisions, writing recommendations.

**Example with Leonardo's case:**
```
Input to Sonnet (about 8,000 tokens):
   - Extracted form data (1,000 tokens)
   - Retrieved policy chunks (5,000 tokens)
   - Coding validation results (2,000 tokens)

Sonnet thinks and writes:
   "Patient meets Gallagher Bassett Workers Comp criterion 2.1
    for E1399 (Iceless Cold Compression):
    ✅ Surgical procedure documented (Sx Date 04/08/2026)
    ✅ Body part is knee (matches policy scope)
    ✅ 60-day rental within payer cap
    Confidence: 0.86"

Output: 500 tokens
```

**Detailed Costing:**

| Type | Price | Example |
|------|-------|---------|
| **Input tokens** | $3.00 per 1M tokens | Reading the case + policy |
| **Output tokens** | $15.00 per 1M tokens | Writing the recommendation |

**Per Case Cost:**
```
Input:  8,000 tokens × $3.00/1M  = $0.024
Output:   500 tokens × $15.00/1M = $0.0075
                                  --------
                          Total: ~$0.031 per Sonnet call

Used ~3 times per case (Supervisor + Policy + Synthesis):
                          ~$0.093 per case
```

**Monthly cost (1,000 cases × $0.093) ≈ $93/month**
*(Table shows $46.50 because many calls are short or cached — heavy synthesis only happens for complex cases)*

---

### 2.3 Amazon Bedrock — Claude Haiku 4.5 — "The Cheap & Fast Worker"

**What it does (simple):**
The lightweight brain. Used for simple tasks where deep thinking isn't needed — classification, date formatting, basic validation.

**Example with Leonardo's case:**
```
Task: "Is this PDF a Letter of Medical Necessity?"
Haiku reads first page → ✅ YES (1 second, $0.0001)

Task: "Convert '04/06/2026' to ISO format"
Haiku → "2026-04-06" (instant, $0.00005)

Task: "Validate that E1399 is a valid HCPCS code format"
Haiku → ✅ Valid format (instant, $0.00005)
```

**Detailed Costing:**

| Type | Price | Why So Cheap |
|------|-------|--------------|
| **Input tokens** | $0.80 per 1M | 5× cheaper than Sonnet |
| **Output tokens** | $4.00 per 1M | 5× cheaper than Sonnet |

**Per Case Cost:**
```
Used by Document Ingestion Agent + Coding Agent
Average: 2,000 input + 200 output tokens per call
Calls per case: ~5

Per case: 5 × (2,000 × $0.80/1M + 200 × $4.00/1M)
        = 5 × ($0.0016 + $0.0008)
        = ~$0.012 per case
```

**Monthly cost (1,000 cases × $0.012) ≈ $12/month**

---

### 2.4 Amazon Titan Text Embeddings V2 — "The Vector Maker"

**What it does (simple):**
Turns words into numbers (vectors) so the computer can compare meaning. When you search "cold compression for knee," Titan converts that into a 1024-number fingerprint.

**Example with Leonardo's case:**
```
Query text: "PA criteria for E1399 under Gallagher Bassett WC in Michigan"
            ↓
Titan converts to vector:
[0.234, -0.567, 0.891, 0.123, ..., 0.456]  (1024 numbers)
            ↓
This vector is used to search the Knowledge Base
for similar-meaning policy chunks
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Per 1,000 input tokens** | $0.00002 |
| **Per query (avg 100 tokens)** | $0.000002 |

**Per Case Cost:**
```
Searches per case: ~5 (Policy + Coding + appeal lookups)
Tokens per search: ~100

Per case: 5 × 100 tokens × $0.00002/1K
        = $0.00001 per case
```

**Monthly cost (1,000 cases) ≈ $0.01/month**
*(Embeddings are almost free — the cost is in the KB infrastructure, not the embedding itself)*

---

### 2.5 Amazon Bedrock Knowledge Bases — "The Smart Library"

**What it does (simple):**
A managed library of all payer policy PDFs. When the Policy Agent asks "What's Gallagher Bassett's rule for E1399?", this service finds the exact page and paragraph.

**Example with Leonardo's case:**
```
Policy Agent's query:
"PA criteria for E1399 cold compression therapy
 under Gallagher Bassett Workers Comp in Michigan"
        ↓
Bedrock KB:
1. Embeds query (uses Titan)
2. Searches OpenSearch vector index (5,000 policies)
3. Filters by metadata {payer: "Gallagher Bassett", state: "MI"}
4. Returns top 5 matching chunks with citations
        ↓
Returns:
   Chunk 1: "GB-WC-DME-v3.2, page 14, criterion 2.1"
            "E1399 approved for post-surgical knee, max 60 days..."
   Chunk 2: "GB-WC-DME-v3.2, page 15, criterion 2.2"
            "Requires MD prescription + diagnosis code..."
```

**Detailed Costing:**

| Component | Price | Why |
|-----------|-------|-----|
| **OpenSearch Serverless** | $0.24 per OCU/hour | Vector index storage + search |
| **Minimum OCUs** | 2 OCUs | AWS requires minimum capacity |
| **Hours/month** | 720 hours | 24/7 operation |
| **Storage** | $0.024/GB/month | Stored embeddings |

**Per Month Cost (Fixed):**
```
2 OCUs × $0.24/hour × 720 hours = $345.60/month
Storage (~1 GB) × $0.024         = $0.024/month
                                   --------
                          Total: ~$346/month
```

**Per Case Cost:**
```
$346 / 1,000 cases  = $0.346 per case (pilot)
$346 / 10,000 cases = $0.034 per case (scale)
```

**This is the biggest fixed cost.** It pays off when volume grows — at 10,000 cases/month, it's only $0.034/case.

---

### 2.6 Amazon Bedrock AgentCore — "The Memory Manager"

**What it does (simple):**
Hosts the agents and keeps their shared notebook (memory). When Document Ingestion finds the codes, AgentCore remembers them so Policy Agent can use them later.

**Example with Leonardo's case:**
```
Step 1: Document Ingestion finishes
        AgentCore saves: {hcpcs: ["E1399", "E0221"], payer: "GB"}

Step 2: Policy Agent starts
        AgentCore gives: {hcpcs: ["E1399", "E0221"], payer: "GB"}
        Policy Agent runs KB search

Step 3: Coding Agent starts
        AgentCore gives: {hcpcs: ["E1399", "E0221"]}
        Coding Agent validates codes

Step 4: Supervisor synthesizes
        AgentCore gives: ALL findings from all agents
        Supervisor writes final recommendation

Bonus: Full reasoning trace saved for audit
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Per agent invocation** | ~$0.015 |
| **Memory storage** | Included |
| **Audit traces** | Included |

**Per Case Cost:**
```
Invocations per case: ~4 (Supervisor + 3 specialists)
Per case: 4 × $0.015 = $0.06 per case
```

**Monthly cost (1,000 cases × $0.06) = $60/month**

---

### 2.7 Amazon S3 — "The Secure Filing Cabinet"

**What it does (simple):**
Stores all PDFs safely. Every LMN, every result, every policy PDF lives in S3. With encryption + WORM (Write Once Read Many), no one can tamper with files.

**Example with Leonardo's case:**
```
1. Reviewer uploads LMN
   → s3://phi-pdf-landing/case-12345/lmn.pdf
   → Encrypted with KMS key
   → Object Lock prevents deletion for 7 years (HIPAA)

2. After processing, result saved:
   → s3://phi-pdf-processed/case-12345/result.json
   → Also encrypted

3. Policy library lives separately:
   → s3://policy-kb/gallagher-bassett/v3.2.pdf
```

**Detailed Costing:**

| Action | Price |
|--------|-------|
| **Storage (Standard)** | $0.023 per GB/month |
| **PUT request** | $0.005 per 1,000 requests |
| **GET request** | $0.0004 per 1,000 requests |
| **KMS encryption call** | $0.03 per 10,000 calls |

**Per Case Cost (avg 2 MB per case):**
```
Storage: 2 MB × $0.023/GB     = $0.00005/month (per case)
Writes:  5 PUTs × $0.005/1K   = $0.000025
Reads:  10 GETs × $0.0004/1K  = $0.000004
                                  --------
                         Total: ~$0.001 per case

Monthly (1,000 cases): ~$1.10/month
```

---

### 2.8 AWS Lambda — "The Serverless Worker"

**What it does (simple):**
Runs your code without managing servers. Each agent is a Lambda function. When something triggers it (like a PDF upload), Lambda runs the agent and shuts down.

**Example with Leonardo's case:**
```
1. LMN uploaded → S3 event triggers EventBridge
2. EventBridge triggers SupervisorLambda
3. SupervisorLambda runs Claude Sonnet (Supervisor agent)
4. Supervisor invokes:
   - DocIngestionLambda (Haiku + Textract)
   - PolicyAgentLambda (Sonnet + Bedrock KB)
   - CodingAgentLambda (Haiku + DynamoDB)
5. All Lambdas run in parallel, each shuts down after work
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Requests** | $0.20 per 1M requests |
| **Compute (GB-second)** | $0.0000166667 per GB-sec |
| **Free tier** | 1M requests + 400,000 GB-sec/month |

**Per Case Cost:**
```
4 Lambda calls × 5 seconds × 1 GB memory each
= 4 × 5 = 20 GB-seconds
= 20 × $0.0000166667 = $0.0003 per case

Plus 4 requests × $0.20/1M = $0.0000008 per case
                                       --------
                              Total: ~$0.0003 per case
                              (essentially free at pilot scale)
```

**Monthly cost (1,000 cases) ≈ $0.00 (within free tier)**

---

### 2.9 Amazon API Gateway — "The Front Door"

**What it does (simple):**
The HTTPS entrance for the Next.js dashboard. When reviewers upload an LMN, the request goes through API Gateway first (handles security, throttling, auth).

**Example with Leonardo's case:**
```
Reviewer clicks "Upload LMN" in dashboard
        ↓
Browser sends POST /upload with file
        ↓
API Gateway:
   ✅ TLS handshake (HTTPS)
   ✅ Cognito token check (is reviewer logged in?)
   ✅ Throttle check (not spamming)
   ✅ Routes to Lambda
        ↓
Lambda saves to S3
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **REST API requests** | $3.50 per 1M requests |
| **Data transfer out** | $0.09 per GB |

**Per Case Cost:**
```
~5 API calls per case (upload + status checks + result fetch)
5 calls × $3.50/1M = $0.0000175 per case

Monthly (1,000 cases × 5 calls = 5,000 calls):
5,000 × $3.50/1M = ~$0.02/month
```

**Monthly cost ≈ $0.01/month**

---

### 2.10 Amazon EventBridge — "The Event Router"

**What it does (simple):**
Listens for events and triggers the right Lambda. When a PDF is uploaded to S3, EventBridge "hears" it and tells the Supervisor Lambda to start.

**Example with Leonardo's case:**
```
S3 PUT event: "case-12345/lmn.pdf uploaded"
        ↓
EventBridge rule matches:
   IF event.source == "s3" AND
      event.bucket == "phi-pdf-landing" AND
      event.key matches "case-*/lmn.pdf"
   THEN trigger SupervisorLambda
        ↓
SupervisorLambda runs with case_id = "case-12345"
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Custom events** | $1.00 per 1M events |
| **AWS service events** | Free |

**Per Case Cost:**
```
~3 events per case (upload, processing, completion)
3 × $1.00/1M = $0.000003 per case

Monthly (1,000 cases × 3 events = 3,000):
3,000 × $1/1M = $0.003/month
```

**Monthly cost ≈ $0.001/month**

---

### 2.11 AWS KMS — "The Key Vault"

**What it does (simple):**
Stores the encryption keys that protect PHI (patient health info). HIPAA requires customer-managed keys — KMS does this safely.

**Example with Leonardo's case:**
```
1. LMN PDF uploaded to S3
        ↓
2. S3 says to KMS: "Encrypt this file"
3. KMS uses our key → encrypts file
4. Stored encrypted in S3

Later:
1. Reviewer requests file
2. S3 says to KMS: "Decrypt this file"
3. KMS uses our key → decrypts file
4. Reviewer sees PDF

Without KMS key, no one can read the file (not even AWS).
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Per key** | $1.00/month |
| **API calls (encrypt/decrypt)** | $0.03 per 10,000 calls |

**Per Case Cost:**
```
Keys used: 3 (S3 PHI, S3 results, CloudWatch logs)
Base: 3 × $1.00 = $3.00/month (fixed)

API calls per case: ~50 (every read/write triggers KMS)
1,000 cases × 50 calls = 50,000 calls
50,000 × $0.03/10K = $0.15/month

Total: ~$3.15/month
```

---

### 2.12 Amazon CloudWatch — "The Logger"

**What it does (simple):**
Collects logs from every agent. When Document Ingestion runs, it writes "Took 2.3s, extracted 12 fields." When something fails, CloudWatch keeps the error.

**Example with Leonardo's case:**
```
Logs for case-12345:
   [10:23:14] SupervisorLambda started for case-12345
   [10:23:15] DocIngestionAgent: extracting fields...
   [10:23:17] DocIngestionAgent: 12 fields extracted (2.3s)
   [10:23:17] PolicyAgent: searching KB...
   [10:23:19] PolicyAgent: 5 chunks retrieved (2.1s)
   [10:23:19] CodingAgent: validating codes...
   [10:23:20] CodingAgent: all codes valid (0.8s)
   [10:23:22] Supervisor: synthesis complete (2.5s)
   [10:23:22] Total case time: 8 seconds
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Logs ingestion** | $0.50 per GB |
| **Logs storage** | $0.03 per GB/month |
| **Custom metrics** | $0.30 per metric/month |

**Per Case Cost:**
```
Log size per case: ~10 KB
1,000 cases = 10 MB = 0.01 GB

Ingestion: 0.01 GB × $0.50  = $0.005/month
Storage:   0.01 GB × $0.03  = $0.0003/month
Plus 30 days of logs accumulating + custom metrics
                                  --------
                          Total: ~$5.50/month
```

---

### 2.13 AWS CloudTrail — "The Audit Logger"

**What it does (simple):**
Records every API call in your AWS account. HIPAA requires you to prove who accessed which PHI when — CloudTrail does this automatically.

**Example with Leonardo's case:**
```
CloudTrail records:
   [10:23:14] reviewer@example.com uploaded case-12345/lmn.pdf
   [10:23:17] SupervisorLambda read case-12345/lmn.pdf
   [10:23:22] SupervisorLambda wrote case-12345/result.json
   [11:45:02] reviewer@example.com read case-12345/result.json

Audit question: "Who accessed Leonardo's LMN today?"
CloudTrail answer: "reviewer@example.com at 10:23:14 and 11:45:02"
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Management events** | Free (first trail) |
| **Data events (S3)** | $0.10 per 100,000 events |
| **Storage in S3** | $0.023/GB |

**Per Case Cost:**
```
Data events per case: ~20 (every S3 read/write)
1,000 cases × 20 events = 20,000 events

Cost: 20,000 × $0.10/100K = $0.02/month
Storage: minimal (~$0.001/month)
                                  --------
                          Total: ~$0.01/month
```

---

### 2.14 AWS Secrets Manager — "The Password Vault"

**What it does (simple):**
Stores passwords, API keys, and tokens safely. When you need the Gallagher Bassett portal token, you ask Secrets Manager — it never sits in code or env vars.

**Example with Leonardo's case:**
```
PolicyAgentLambda needs to call payer portal:

❌ BAD: hardcoded API key in code
       api_key = "abc123secret456"  # exposed!

✅ GOOD: fetch from Secrets Manager
       secret = secrets_manager.get_secret_value(
           SecretId='gallagher-bassett-portal'
       )
       api_key = secret['api_key']

Secrets Manager:
   ✅ Encrypted with KMS
   ✅ Auto-rotates every 90 days
   ✅ IAM-scoped (only PolicyAgent can read)
   ✅ All access logged in CloudTrail
```

**Detailed Costing:**

| Metric | Price |
|--------|-------|
| **Per secret** | $0.40/month |
| **API calls** | $0.05 per 10,000 calls |

**Per Case Cost:**
```
Secrets stored: ~3 (payer tokens, NPI registry, etc.)
Base: 3 × $0.40 = $1.20/month (fixed)

API calls per case: ~2
1,000 cases × 2 = 2,000 calls
Cost: 2,000 × $0.05/10K = $0.01/month
                                  --------
                          Total: ~$1.20/month
```

---

### 2.15 Total Per-Case Cost Summary

Putting it all together for one case (Leonardo's case):

| Service | Per-Case Cost |
|---------|--------------|
| Textract (1 page, full features) | $0.080 |
| Sonnet 4.6 (3 calls) | $0.093 |
| Haiku 4.5 (5 calls) | $0.012 |
| Titan Embeddings | $0.00001 |
| Bedrock KB (amortized at 1K cases) | $0.346 |
| AgentCore (4 invocations) | $0.060 |
| S3 storage + I/O | $0.001 |
| Lambda | $0.0003 |
| API Gateway | $0.00002 |
| EventBridge | $0.000003 |
| KMS | $0.003 |
| CloudWatch | $0.006 |
| CloudTrail | $0.00001 |
| Secrets Manager | $0.001 |
| **TOTAL PER CASE** | **~$0.60** |

**Monthly cost for 1,000 cases: ~$600/month**
*(The KB infrastructure is fixed at $346, so the cost-per-case drops sharply as volume grows — at 10K cases/month, the per-case cost drops to ~$0.29.)*

---

## 3. Multi-Agent Architecture

```
                     ┌─────────────────────────────────────┐
                     │   SUPERVISOR (Orchestrator)         │
                     │   Model:  Claude Sonnet 4.6         │
                     │   Runtime: Bedrock AgentCore        │
                     │   Job:    Plan → fan-out → synthesize│
                     └──────────────────┬──────────────────┘                 
                                        │ 
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
   ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
   │ Document Ingestion │  │ Policy/PA-Criteria │  │   Coding Agent     │
   │ Model: Haiku 4.5   │  │ Model: Sonnet 4.6  │  │ Model: Haiku 4.5   │
   │ Tool:  Textract    │  │ Tool:  Bedrock KB  │  │ Tool:  DynamoDB    │
   │        (Forms+QnA) │  │        (kb.retrieve│  │        crosswalks  │
   │                    │  │         + Titan V2)│  │        (CPT/ICD-10)│
   └────────────────────┘  └────────────────────┘  └────────────────────┘
```

### 3.1 Document Ingestion Agent
- **One-line job:** Extract every field and checkbox from the LMN PDF.
- **Model:** Claude Haiku 4.5 (cheap normalization of Textract output into our schema)
- **Tools:** `textract.analyze_document` with `FeatureTypes=[FORMS, QUERIES, TABLES]`
- **Inputs:** S3 URI of the LMN PDF
- **Outputs:** `{patient, insurance, codes, body_part, laterality, rental_duration, surgical_flag, provider}` JSON
- **IAM scope:** `s3:GetObject` on `phi-pdf-landing/`, `textract:AnalyzeDocument`

### 3.2 Policy / PA-Criteria Agent
- **One-line job:** Find the exact payer rulebook for the requested HCPCS codes.
- **Model:** Claude Sonnet 4.6 (semantic understanding of policy prose)
- **Tools:** `kb.retrieve` against the `policy-kb` Bedrock Knowledge Base (Titan V2 embeddings, OpenSearch Serverless backend)
- **Inputs:** `{payer_name, hcpcs_codes[], state}`
- **Outputs:** Ordered checklist of payer criteria, each with a page-anchored citation back to the source policy PDF
- **IAM scope:** `bedrock:Retrieve` on `policy-kb`

### 3.3 Coding Agent
- **One-line job:** Validate every code combination + modifier + NCCI compatibility.
- **Model:** Claude Haiku 4.5 (rules-based validation, no deep reasoning needed)
- **Tools:** `validate_cpt`, `validate_icd10`, `crosswalk_hcpcs`, `lookup_carc_rarc` — all backed by DynamoDB lookup tables
- **Inputs:** Extracted codes from Document Ingestion + criteria from Policy Agent
- **Outputs:** `{validated_codes[], modifier_recommendations[], coding_issues[]}`
- **IAM scope:** `dynamodb:GetItem` on `cpt-table`, `icd10-table`, `ncci-table`

### 3.4 Supervisor (Orchestrator)
- **One-line job:** Coordinate the 3 specialists, detect gaps, write the final answer.
- **Model:** Claude Sonnet 4.6
- **Runtime:** Bedrock AgentCore (provides session memory + reasoning traces)
- **Inputs:** New case S3 manifest
- **Outputs:** `{pa_recommendation, required_documents, open_gaps, confidence}`
- **IAM scope:** `bedrock:InvokeAgent` on the 3 specialist agents

---

## 4. End-to-End Technical Flow (Stage by Stage)

```
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 1 — INTAKE                                                    │
│ Reviewer uploads LMN PDF via Next.js dashboard                      │
│   • API Gateway (REST + TLS)                                         │
│   • Lambda (FastAPI handler)                                         │
│   • Cognito (user auth)                                              │
│   • S3 PUT → s3://phi-pdf-landing/case-{id}/lmn.pdf  [KMS-encrypted]│
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 2 — TRIGGER                                                    │
│   • S3:ObjectCreated event                                           │
│   • EventBridge rule routes to SupervisorLambda                      │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 3 — SUPERVISOR INIT (Bedrock AgentCore)                       │
│   • AgentCore loads Supervisor agent + session memory                │
│   • Model: Claude Sonnet 4.6                                         │
│   • Builds initial plan from S3 manifest                             │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 4 — DOCUMENT INGESTION (sequential — must finish first)       │
│   • Agent: Document Ingestion Agent (Haiku 4.5)                      │
│   • Tool:  textract.analyze_document(FORMS, QUERIES, TABLES)         │
│   • Service: Amazon Textract                                         │
│   • Extracts:                                                        │
│       icd10_codes      = ["M75.82"]                                  │
│       hcpcs_codes      = ["E1399", "E0221"]                          │
│       body_part        = "Knee"                                      │
│       laterality       = "Left"                                      │
│       rental_duration  = 60 (days)                                   │
│       surgical_flag    = "Y"                                         │
│       payer            = "Gallagher Bassett"                         │
│       claim_number     = "011202-00983296"                           │
│       provider_npi     = "1225091143"                                │
│       signature_present = true                                       │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 5 — PARALLEL FAN-OUT                                           │
│                                                                      │
│  ┌──────────────────────────┐    ┌──────────────────────────┐       │
│  │ Policy/PA-Criteria Agent │    │ Coding Agent             │       │
│  │ (Sonnet 4.6)             │    │ (Haiku 4.5)              │       │
│  │                          │    │                          │       │
│  │ • kb.retrieve(query=     │    │ • validate_cpt(E1399)    │       │
│  │   "PA criteria for E1399 │    │ • validate_icd10(M75.82) │       │
│  │    + Gallagher Bassett   │    │ • crosswalk_hcpcs        │       │
│  │    WC + Florida")        │    │   ([E1399, E0221])       │       │
│  │ • Bedrock KB             │    │ • DynamoDB lookups       │       │
│  │   (Titan V2 + OpenSearch)│    │                          │       │
│  │                          │    │ Returns:                 │       │
│  │ Returns:                 │    │  validated_codes[],      │       │
│  │  criteria[] + page cites │    │  modifier_recs[],        │       │
│  └──────────────────────────┘    │  coding_issues[]         │       │
│                                  └──────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 6 — SUPERVISOR EVALUATION (Dynamic Outcomes)                  │
│   • AgentCore working memory holds all specialist outputs            │
│   • Supervisor (Sonnet 4.6) checks:                                  │
│       - Are all policy criteria addressed by extracted form data?    │
│       - surgical_flag=Y but no op-note uploaded? → open_gaps         │
│       - Any coding issues from Coding Agent? → flag                  │
│   • If gaps remain → re-dispatch a specialist with a narrowed query  │
│     (e.g. "Policy Agent, find criteria specific to 60-day rental")   │
│   • Loop until criteria are addressed or marked unmet-and-unattainable│
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 7 — SYNTHESIS                                                  │
│   • Supervisor (Sonnet 4.6) writes the final structured output:      │
│                                                                      │
│   {                                                                  │
│     "case_id": "case-12345",                                         │
│     "pa_recommendation": {                                           │
│       "argument": "Patient (Workers Comp claim 011202-00983296)      │
│                    meets Gallagher Bassett criteria for E1399        │
│                    (Iceless Cold Compression) and E0221 (LLLT)       │
│                    post-surgical knee, ICD-10 M75.82, 60-day rental",│
│       "policy_citations": [                                          │
│         { "policy": "GB-WC-DME-v3.2", "page": 14, "criterion": "2.1"}│
│       ],                                                             │
│       "supporting_codes": {                                          │
│         "hcpcs":  ["E1399", "E0221"],                                │
│         "icd10":  ["M75.82"],                                        │
│         "modifiers": ["LT", "RR"]                                    │
│       },                                                             │
│       "confidence": 0.86                                             │
│     },                                                               │
│     "required_documents": [                                          │
│       "Completed LMN (uploaded ✓)",                                  │
│       "Operative note from 04/08/2026 surgery",                      │
│       "Product invoice / MSRP sheet"                                 │
│     ],                                                               │
│     "open_gaps": [                                                   │
│       "Surgical Patient = Y but no operative note uploaded — request│
│        from Dr. Bridges before submission"                           │
│     ]                                                                │
│   }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STAGE 8 — DELIVERY + AUDIT                                           │
│   • Output written to s3://phi-pdf-processed/case-{id}/result.json   │
│   • Displayed in Next.js dashboard via API Gateway                   │
│   • Every agent tool call → CloudWatch Logs (case_id correlated)     │
│   • Every PHI access → CloudTrail data event                         │
│   • Full reasoning trace → AgentCore traces (HIPAA audit)            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Cost Summary

| Tier | Volume | Monthly Cost | Per Case |
|---|---|---|---|
| Pilot | 1,000 cases | **~$562** | $0.56 |
| Growth | 5,000 cases | ~$1,426 | $0.29 |
| Scale | 10,000 cases | ~$2,852 | $0.29 |
| Nationwide | 50,000 cases | ~$11,838 | $0.24 |

**Top cost driver:** Bedrock KB / OpenSearch Serverless minimum (~$346/mo fixed). Becomes negligible past 5,000 cases/month.

---

## 6. Prerequisites

- Signed **AWS BAA** (HIPAA contract — required before any PHI lands in the account)
- **Bedrock model access** in `us-east-1` or `us-west-2` for: `anthropic.claude-sonnet-4-6`, `anthropic.claude-haiku-4-5`, `amazon.titan-embed-text-v2:0`
- **S3 buckets** with KMS-CMK encryption + Object Lock (WORM retention)
- **Per-agent IAM roles** (least-privilege scoping)
- **Bedrock Knowledge Base** over `s3://policy-kb/` ingested with payer policy PDFs
- **DynamoDB tables**: `cpt-table`, `icd10-table`, `ncci-table`, `carc-rarc-table` (loaded from X12 + CMS reference data)
- **Local toolchain**: Python ≥ 3.11 with `strands-agents`, `boto3`, `fastapi`; Node ≥ 20 for Next.js dashboard

---

## 7. Failure Modes & Fallbacks

| Failure | Fallback |
|---|---|
| Textract cannot read a checkbox or field | Route case to manual-review queue with PDF attached |
| Required field missing on LMN (e.g. ICD-10 left blank) | Surface in `open_gaps`; never invent values |
| Policy Agent retrieves no matching chunks | Supervisor flags `payer_policy_unknown` and pauses for human triage |
| Supervisor loops > 5 iterations without progress | Stop; emit current state; hand to human reviewer |
| Bedrock throttling | Token-bucket retry via `tenacity`; failover to secondary region |

---

## 8. Sample Input Script

The LMN PDF used as input has the following structure (this is the only document the system receives per case):

```
Patient Demographics:
  Patient Name:    Leonardo Mesa Verdecia
  DoB:             08/14/1964
  DoI:             01/03/2025
  Workers Comp:    ✅
  Surgical Patient: Y ✅
  Sx Date:         04/08/2026

Insurance:
  Insurance Carrier: Gallagher Bassett
  Claim #:           011202-00983296
  Adjuster Name:     Melissa Plourde
  Adjuster Phone:    239 271 3027

ICD-10 Code(s):  M75.82

Therapy Products (checked):
  ✅ Iceless Intermittent Cold Compression Therapy [HCPCS: E1399]
       Rental Duration: ✅ 60 Days
       Laterality:      ✅ Left
       Body Part:       ✅ Knee
  ✅ Photobiomodulation Therapy (LLLT) [HCPCS: E0221]
       Laterality:      ✅ Left
       Body Part:       ✅ Shoulder

Provider:
  Provider Name:        Dr. Mark Bridges, M.D.
  NPI:                  1225091143
  Provider Signature:   [signed]
  Date:                 04/06/2026
```

Every value above is either a labeled text field or a checkbox — **all extractable by Textract alone**. There is no free-text clinical narrative on this form, which is why Comprehend Medical and HealthLake are not part of the architecture.
