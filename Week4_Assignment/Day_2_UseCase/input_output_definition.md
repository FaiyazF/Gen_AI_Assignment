# Input & Output Definition – AI Duplicate Bug Detection System

------------------------------------------------------------------------

## 1. Input Sources

### 1.1 Primary Data Source – MongoDB Atlas (`bugs` Collection)

The primary input is the existing bug database stored in MongoDB Atlas.

**Collection:** `bugs`

**Document fields consumed:**

| Field               | Type        | Used For                        |
|---------------------|-------------|---------------------------------|
| `Bug ID`            | String      | Unique identifier (`BUG-0001`)  |
| `Bug Title`         | String      | Search, embedding, display      |
| `Module`            | String      | Search, filtering, embedding    |
| `Module Wise`       | String      | Categorization, filtering       |
| `Steps`             | String      | Search, embedding               |
| `Expected Results`  | String      | Search, embedding               |
| `Actual Results`    | String      | Search, embedding               |
| `Acceptance Criteria`| String     | Display, context                |
| `text`              | String      | BM25 keyword search index       |
| `embedding`         | Array[1024] | Vector similarity search         |

**Sample document:**

    {
      "_id": { "$oid": "69b67ccec0173257fb08ab74" },
      "Bug ID": "BUG-0001",
      "Bug Title": "Funds transfer not updating balance #0",
      "Module": "ATM Services",
      "Module Wise": "ATM Services Wise",
      "Steps": "Step 1: Insert card into ATM -> Step 2: Enter PIN -> ...",
      "Expected Results": "ATM PIN change should be processed successfully...",
      "Actual Results": "System failed to perform as expected",
      "Acceptance Criteria": "Operation must meet functional and business requirements",
      "text": "",
      "embedding": [ -0.0409, 0.0267, 0.0330, ... ]
    }

------------------------------------------------------------------------

### 1.2 Seed Data File – CSV

**File:** `src/data/banking_bugs.csv`

Used for initial data loading into MongoDB.

**CSV columns:**

    Bug ID, Bug Title, Steps, Module, Module Wise, Expected Results, Actual Results, Acceptance Criteria

**Sample row:**

    BUG-0001,Funds transfer not updating balance #0,Step 1: Insert card into ATM -> Step 2: Enter PIN -> ...,ATM Services,ATM Services Wise,ATM PIN change should be processed successfully...,System failed to perform as expected,Operation must meet functional and business requirements

**Banking modules covered in seed data:**

- ATM Services
- Authentication
- Account Management
- Mobile Banking
- Statements
- Notifications

------------------------------------------------------------------------

### 1.3 Index Configuration Files

**BM25 Text Index:** `src/config/bankingbugs-bm25-index.json`

Defines MongoDB Atlas Search index for keyword matching.

- Fields indexed: `bugId`, `bugTitle`, `steps`, `module`, `moduleWise`, `expectedResults`, `actualResults`, `acceptanceCriteria`
- Analyzers: `lucene.standard` (text fields), `lucene.keyword` (ID/category fields)

**Vector Search Index:** `src/config/bankingbugs-vector-index.json`

Defines MongoDB Atlas Vector Search index.

- Vector field: `embedding` (1024 dimensions, cosine similarity)
- Filter fields: `bugId`, `bugTitle`, `steps`, `module`, `moduleWise`, `expectedResults`, `actualResults`, `acceptanceCriteria`

------------------------------------------------------------------------

### 1.4 User Input (API Requests)

Users or agents provide input through REST API calls:

**Search input:**

    {
      "query": "transfer not updating balance",
      "topK": 10,
      "offset": 0
    }

**Bug creation input:**

    {
      "Bug Title": "ATM transfer not updating balance",
      "Module": "ATM Services",
      "Steps": "Step 1: Insert card -> Step 2: Enter PIN -> Step 3: Transfer funds",
      "Expected Results": "Balance should update after transfer",
      "Actual Results": "Balance unchanged",
      "Acceptance Criteria": "Transfer must update account balance"
    }

**Duplicate check input:**

    {
      "title": "ATM transfer not updating balance",
      "steps": "Insert card enter pin transfer"
    }

------------------------------------------------------------------------

### 1.5 External AI Services (Input to System)

| Service            | Provider              | Input Sent                    | Output Received              |
|--------------------|-----------------------|-------------------------------|------------------------------|
| Embedding API      | Mistral (`mistral-embed`) | Bug text (concatenated fields) | 1024-dimension float vector  |
| LLM Re-ranking    | Groq / OpenAI / Mistral | Query bug + candidate bugs    | Ranked list with scores + reasoning |

------------------------------------------------------------------------

### 1.6 Environment Configuration

| Variable              | Purpose                              |
|-----------------------|--------------------------------------|
| `MONGODB_URI`         | MongoDB Atlas connection string      |
| `MONGODB_DB_NAME`     | Database name                        |
| `MISTRAL_API_KEY`     | Embedding model API key              |
| `LLM_API_KEY`         | LLM service API key (Groq/OpenAI)   |
| `LLM_MODEL`           | LLM model name for re-ranking       |
| `PORT`                | Express server port                  |
| `API_KEY`             | API key for endpoint authentication  |

------------------------------------------------------------------------

## 2. Output Produced by the Agent

### 2.1 Similar Bug Search Results

**Endpoint:** `POST /v1/bugs/similar`

    {
      "results": [
        {
          "bugId": "BUG-0001",
          "title": "Funds transfer not updating balance #0",
          "similarityScore": 92,
          "confidence": "High"
        },
        {
          "bugId": "BUG-0007",
          "title": "Funds transfer not updating balance #6",
          "similarityScore": 78,
          "confidence": "Medium"
        }
      ],
      "total": 25,
      "offset": 0,
      "limit": 10
    }

------------------------------------------------------------------------

### 2.2 Duplicate Detection Results

**Endpoint:** `POST /v1/bugs/check-duplicate`

    {
      "duplicates": [
        {
          "bugId": "BUG-0001",
          "title": "Funds transfer not updating balance #0",
          "similarityScore": 88,
          "confidence": "High",
          "matchingFields": ["module", "expectedResult", "steps"],
          "reasoning": "Same module with identical failure scenario on fund transfer"
        }
      ],
      "possibleDuplicates": [
        {
          "bugId": "BUG-0205",
          "title": "Balance not refreshed after transfer",
          "similarityScore": 55,
          "confidence": "Low",
          "matchingFields": ["module"]
        }
      ],
      "recommendation": "duplicate"
    }

**Recommendation values:**

| Value             | Meaning                             |
|-------------------|-------------------------------------|
| `"duplicate"`     | Strong duplicate found (score ≥ 70) |
| `"possible_duplicate"` | Related bug exists (score 40–69) |
| `"create_new_bug"` | Safe to create (score < 40)        |

------------------------------------------------------------------------

### 2.3 Bug Creation Confirmation

**Endpoint:** `POST /v1/bugs`

    {
      "message": "Bug created successfully",
      "bugId": "BUG-0042"
    }

**Side effects on creation:**

- `text` field auto-generated (concatenation of title + module + steps + expected + actual)
- `embedding` vector auto-generated via Mistral API
- `Bug ID` auto-incremented from counters collection
- `Module Wise` derived from Module field

------------------------------------------------------------------------

### 2.4 Bug Details

**Endpoint:** `GET /v1/bugs/{bugId}`

    {
      "Bug ID": "BUG-0001",
      "Bug Title": "Funds transfer not updating balance #0",
      "Module": "ATM Services",
      "Module Wise": "ATM Services Wise",
      "Steps": "Step 1: Insert card into ATM -> Step 2: Enter PIN -> ...",
      "Expected Results": "ATM PIN change should be processed successfully...",
      "Actual Results": "System failed to perform as expected",
      "Acceptance Criteria": "Operation must meet functional and business requirements"
    }

------------------------------------------------------------------------

### 2.5 LLM Re-Ranking Output

**Endpoint:** `POST /v1/search/rerank`

    {
      "reranked": [
        {
          "bugId": "BUG-0101",
          "finalScore": 92,
          "reasoning": "Same module, identical OTP failure path during withdrawal"
        },
        {
          "bugId": "BUG-0205",
          "finalScore": 48,
          "reasoning": "Same module but different failure mechanism (timeout vs OTP)"
        }
      ]
    }

------------------------------------------------------------------------

### 2.6 Health Check Output

**Endpoint:** `GET /v1/health`

    {
      "status": "ok",
      "uptime": 84320
    }

**Endpoint:** `GET /v1/health/db`

    {
      "status": "connected",
      "database": "banking_bugs",
      "collections": ["bugs", "counters"]
    }

------------------------------------------------------------------------

### 2.7 Structured Logs (System Output)

Written to stdout/log files for observability:

    {
      "requestId": "req-123",
      "endpoint": "/v1/bugs/similar",
      "durationMs": 1500,
      "statusCode": 200,
      "componentTimings": {
        "embeddingMs": 180,
        "bm25Ms": 290,
        "vectorMs": 310,
        "rerankMs": 520
      }
    }

------------------------------------------------------------------------

## 3. Input → Output Flow Summary

    ┌─────────────────────────────┐
    │       INPUT SOURCES         │
    ├─────────────────────────────┤
    │ • MongoDB bugs collection   │
    │ • banking_bugs.csv (seed)   │
    │ • BM25 index config (JSON)  │
    │ • Vector index config (JSON)│
    │ • User API requests         │
    │ • Environment variables     │
    │ • Mistral Embedding API     │
    │ • LLM API (Groq/OpenAI)    │
    └──────────┬──────────────────┘
               │
               ▼
    ┌─────────────────────────────┐
    │     PROCESSING PIPELINE     │
    ├─────────────────────────────┤
    │ 1. Parse user input         │
    │ 2. Generate embedding       │
    │ 3. BM25 keyword search      │
    │ 4. Vector similarity search │
    │ 5. Merge & deduplicate      │
    │ 6. LLM re-ranking          │
    │ 7. Score & classify         │
    └──────────┬──────────────────┘
               │
               ▼
    ┌─────────────────────────────┐
    │       OUTPUT PRODUCED       │
    ├─────────────────────────────┤
    │ • Ranked similar bugs       │
    │ • Duplicate detection result│
    │ • Recommendation action     │
    │ • Bug creation confirmation │
    │ • Bug details               │
    │ • LLM reasoning/explanation │
    │ • Structured JSON logs      │
    │ • Health status             │
    └─────────────────────────────┘

------------------------------------------------------------------------

*Document Version: 1.0*
*Date: March 15, 2026*
*Project: RAG-based Duplicate Bug Detection System*
