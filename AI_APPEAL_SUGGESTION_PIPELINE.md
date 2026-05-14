# AI Appeal Suggestion Pipeline — Complete Architecture Guide

## The Scenario

    → We analyze 1 year of historical data (denials + acceptances)
    → Train a RAG system on that data
    → When a new denial comes in, AI suggests HOW to re-appeal based on past successful patterns


## Tech Stack

| Component | Technology | Purpose |
|---|---|---|
| **Vector Database** | Pinecone (Serverless) | Store embedded chunks of historical claims |
| **LLM** | Claude Sonnet 4.6 via AWS Bedrock | Generate appeal suggestions |
| **Embeddings** | Amazon Titan Text Embeddings V2 via Bedrock | Convert text → vectors |
| **Backend** | FastAPI (Python) | Orchestrate the pipeline |
| **Frontend** | Next.js | Display suggestions to client |

### Model IDs (AWS Bedrock)

| Model | Bedrock Model ID | Specs |
|---|---|---|
| Claude Sonnet 4.6 | `anthropic.claude-sonnet-4-6` | 1M token context, 64K output, launched Feb 2026 |
| Titan Embeddings V2 | `amazon.titan-embed-text-v2:0` | 8,192 tokens input, 1024 dimensions output |


## 1. Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ONE-TIME DATA INGESTION (1 year of history)         │
│                                                                         │
│   Client's PDFs (835 EORs, denial letters, appeal letters, notes)      │
│       │                                                                 │
│       ▼                                                                 │
│   PDF EXTRACTION (PyMuPDF / pdfplumber / AWS Textract)                 │
│       │                                                                 │
│       ▼                                                                 │
│   DESTRUCTURE — Extract only:                                          │
│       • Denial reason codes (CARC/RARC)                                │
│       • Insurance company notes                                        │
│       • Procedure codes (CPT) that were denied                         │
│       • Amount billed vs amount paid                                   │
│       • Appeal outcome (if re-appeal was successful)                   │
│       │                                                                 │
│       ▼                                                                 │
│   CHUNK into logical segments (1 chunk = 1 claim denial case)          │
│       │                                                                 │
│       ▼                                                                 │
│   EMBED via Amazon Titan Text Embeddings V2 (Bedrock)                  │
│       │                                                                 │
│       ▼                                                                 │
│   STORE in Pinecone (vector + metadata)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     REAL-TIME QUERY (when new denial arrives)           │
│                                                                         │
│   New 835 payment arrives (via DaisyBill webhook: bill_payment.created)│
│       │                                                                 │
│       ▼                                                                 │
│   EXTRACT denial reasons + codes from the new 835                      │
│       │                                                                 │
│       ▼                                                                 │
│   EMBED the denial query via Titan Embeddings V2                       │
│       │                                                                 │
│       ▼                                                                 │
│   SEARCH Pinecone for top-K similar historical cases                   │
│       │                                                                 │
│       ▼                                                                 │
│   RETRIEVE matching chunks (past denials + their outcomes)             │
│       │                                                                 │
│       ▼                                                                 │
│   PROMPT Claude Sonnet 4.6 with:                                       │
│       • The current denial details                                     │
│       • Retrieved similar past cases                                   │
│       • Past successful appeal strategies                              │
│       │                                                                 │
│       ▼                                                                 │
│   Claude generates: APPEAL SUGGESTION                                  │
│       • What to argue                                                  │
│       • Which codes to reference                                       │
│       • What documentation to include                                  │
│       • Confidence score based on historical success rate               │
│       │                                                                 │
│       ▼                                                                 │
│   Display in Next.js dashboard → Client reviews → Approves appeal      │
│       │                                                                 │
│       ▼                                                                 │
│   POST /bills/{id}/request_for_second_review (DaisyBill API)           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Sources — What Comes From PDFs

The client provides **1 year of historical data** in PDF format. These PDFs typically include:

### Key Codes in the PDFs

**CARC — Claim Adjustment Reason Codes** (why insurance denied/adjusted):
```
Code    Meaning
----    -------
CO-4    The procedure code is inconsistent with the modifier used
CO-16   Claim/service lacks information needed for adjudication
CO-18   Exact duplicate claim/service
CO-29   The time limit for filing has expired
CO-50   Non-covered service (not a benefit of the plan)
CO-96   Non-covered charge(s)
CO-97   Payment adjusted: benefit for this service is in another claim
PR-1    Deductible amount
PR-2    Coinsurance amount
PR-3    Co-payment amount
OA-23   Payment adjusted: charges were paid by another payer
```

**RARC — Remittance Advice Remark Codes** (supplementary explanation):
```
Code     Meaning
----     -------
N30      Patient ineligible on dates of service
N56      Procedure code not valid for the date of service
M15      Separately billed services/tests denied
N362     Missing/Incomplete documentation
N657     This should be billed to a different payer
```

> Source: https://x12.org/codes/claim-adjustment-reason-codes

---

## 3. PDF Extraction Pipeline

### 3.1 Libraries for PDF Parsing

| Library | Best For | Install |
|---|---|---|
| **pdfplumber** | Text-heavy PDFs with tables (835 EORs) | `pip install pdfplumber` |
| **PyMuPDF (fitz)** | Fast general-purpose PDF text extraction | `pip install PyMuPDF` |
| **AWS Textract** | Scanned PDFs / images / handwritten notes | AWS service (API call) |
| **pdf2image + Tesseract** | OCR fallback for scanned docs | `pip install pdf2image pytesseract` |


## 4. Data Destructuring — What to Extract

You do NOT want to embed entire PDFs into Pinecone. You need to **extract only the relevant fields** for denial analysis.


## 5. Chunking Strategy

### 5.1 One Chunk = One Claim Denial Case

Each vector in Pinecone represents **one complete denial case** — not random text fragments.

```
┌──────────────────────────────────────────────────┐
│ CHUNK (one vector in Pinecone)                   │
│                                                  │
│ Denial Case: WC2024-001                          │
│ Payer: State Fund Insurance                      │
│ CPT: 99213, 97140                                │
│ CARC: CO-16, CO-50                               │
│ RARC: N362                                       │
│ Denial Reason: "Missing documentation for        │
│   medical necessity of physical therapy"          │
│ Insurance Notes: "Submit operative report and     │
│   treatment plan to support medical necessity"    │
│ Billed: $450.00 | Paid: $0.00                    │
│                                                  │
│ --- APPEAL INFO (if available) ---               │
│ Was Appealed: YES                                │
│ Strategy: "Submitted physician's letter + PT     │
│   treatment plan documenting medical necessity"   │
│ Outcome: APPROVED — full payment $450.00         │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 5.2 Why This Chunking Strategy?

| Strategy | Pros | Cons |
|---|---|---|
| **One chunk = one denial case** ✅ | Each result is a complete, actionable case | Chunks vary in size |
| One chunk = one page | Consistent size | Splits cases across chunks, loses context |
| Fixed 500-token chunks | Standard RAG approach | Breaks denial reasoning across chunks |


> **Titan Embeddings V2 Limit**: Max 8,192 tokens (~38,500 characters). Each denial case
> chunk should be well under this limit (typically 500-2000 characters).

---

## 6. Embedding Pipeline

### 6.1 Amazon Titan Text Embeddings V2

| Property | Value |
|---|---|
| **Model ID** | `amazon.titan-embed-text-v2:0` |
| **Max Input** | 8,192 tokens (~50,000 characters) |
| **Output Dimensions** | 1,024 (default), 512, 256 |
| **Recommended Dimension** | **1,024** (best accuracy for RAG) |
| **Throttling** | Requests Per Minute (RPM), not tokens |

### 6.3 How Embedding Works (Conceptually)

```
"Denial Reason: Missing documentation for       ─────►  [0.023, -0.156, 0.891, ..., 0.045]
 medical necessity of physical therapy.                   (1024 float numbers)
 CARC: CO-16. RARC: N362"

"Denial Reason: Service not medically necessary  ─────►  [0.021, -0.148, 0.887, ..., 0.041]
 for the diagnosis provided. CARC: CO-50"                 (1024 float numbers)
                                                          ↑ Very similar vectors!
                                                            (close in vector space)
```

When a new denial comes in, its embedding will be **close in vector space** to similar past denials, allowing Pinecone to find the best matches.



## 8. Training / Indexing the Data

> This is a **RAG (Retrieval-Augmented Generation)** approach:
> - "Training" = Indexing historical data into Pinecone
> - "Inference" = Retrieving similar cases + Claude generates suggestions


### 8.2 Training Pipeline Visualization
```
1 YEAR OF CLIENT PDFs
        │
        ▼
┌─────────────────┐
│  PDF EXTRACTION  │  pdfplumber / PyMuPDF / Textract
│  (raw text)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CLASSIFY TYPE   │  835_eor? denial_letter? appeal_letter?
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  DESTRUCTURE     │  Extract CARC, RARC, denial reason,
│  (extract fields)│  CPT codes, amounts, notes
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LINK RECORDS    │  Match denial → appeal outcome
│  (by claim #)    │  by same claim number
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  FORMAT CHUNK    │  denial_record_to_chunk_text()
│  (text for embed)│  One text blob per denial case
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  EMBED           │  Amazon Titan Text Embeddings V2
│  (text → vector) │  → [1024-dim float vector]
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  STORE IN        │  Pinecone Serverless
│  PINECONE        │  vector + metadata
└─────────────────┘
```

---

## 9. Query Flow — Generating Suggestions

### 9.1 When a New Denial Arrives

```
DaisyBill Webhook: bill_payment.created
    │
    ├─ Extract denial info from the new 835 
    │
    ├─ Format as query text
    │
    ├─ Embed query with Titan Embeddings V2
    │
    ├─ Search Pinecone for top-10 similar past denials
    │
    ├─ Build prompt with:
    │   • Current denial details
    │   • Top-10 similar past cases (with outcomes)
    │
    ├─ Send to Claude Sonnet 4.6 via Bedrock
    │
    └─ Claude generates structured appeal suggestion
```


## 11. Cost Estimates

### Pinecone (Serverless)

| Tier | Cost | What You Get |
|---|---|---|
| **Free** | $0 | 100K vectors, 1 index, 2GB storage |
| **Standard** | ~$0.33/1M reads | Pay as you go |

> For 1 year of data (~5,000-10,000 denial cases), the **free tier is sufficient** for development.

### AWS Bedrock — Titan Embeddings V2

| Usage | Cost |
|---|---|
| Input | $0.00002 / 1K tokens |
| 10,000 denial records (~500 tokens each) | ~$0.10 total |

> Embedding is extremely cheap — your entire training run costs **cents**.

### AWS Bedrock — Claude Sonnet 4.6

| Usage | Cost |
|---|---|
| Input tokens | $0.003 / 1K tokens |
| Output tokens | $0.015 / 1K tokens |
| 1 suggestion (~3K input + 1K output) | ~$0.024 per suggestion |
| 100 suggestions/month | ~$2.40/month |

> Very affordable for the value delivered.

### Total Estimated Monthly Cost (Production)

| Component | Monthly Cost |
|---|---|
| Pinecone Serverless (free tier) | $0 |
| Titan Embeddings (queries) | ~$0.50 |
| Claude Sonnet 4.6 (suggestions) | ~$5-25 |
| AWS infrastructure | Varies |
| **Total** | **~$5-30/month** |
