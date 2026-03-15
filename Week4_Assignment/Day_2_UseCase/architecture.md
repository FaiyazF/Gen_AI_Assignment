# AI Duplicate Bug Detection & Bug Creation System Architecture

## 1. System Overview

This system helps QA engineers and AI agents:

-   Search similar bugs
-   Identify duplicate bugs automatically
-   View bug details
-   Create new bugs safely
-   Prevent duplicate defects in the system

The system uses **Hybrid AI Search**:

-   BM25 keyword search
-   Vector similarity search
-   LLM semantic re-ranking

Tech Stack:

-   Node.js
-   TypeScript
-   Express.js
-   MongoDB Atlas
-   Vector Embeddings
-   LLM (Groq / OpenAI / Mistral)

Performance Targets:

-   End-to-end duplicate detection: 2–5 seconds
-   Embedding generation: < 500ms
-   Search (BM25 + vector): < 1s
-   LLM re-ranking: < 2s

------------------------------------------------------------------------

# 2. Core Capabilities

## Bug Discovery

Users or agents can search existing bugs using:

-   Bug Title
-   Expected Results
-   Steps
-   Free text query

## Duplicate Detection

When creating a bug the system checks for similar bugs.

## Bug Details

Users can open any bug from search results.

## Bug Creation

Users can create a new bug if no strong duplicates exist.

------------------------------------------------------------------------

# 3. User Workflow

## Scenario 1 --- Search Similar Bugs

1.  User enters title or description
2.  System searches existing bugs
3.  System returns ranked results
4.  User selects bug
5.  Bug details are opened

------------------------------------------------------------------------

## Scenario 2 --- Create New Bug

1.  User clicks **Create New Bug**
2.  System displays bug form

Fields:

-   Bug Title
-   Module
-   Module Wise
-   Steps to Reproduce
-   Expected Results
-   Actual Results
-   Acceptance Criteria

3.  User enters **Bug Title**
4.  User clicks **Check Similar Bugs**
5.  System displays similar bugs

Decision logic:

| Score   | Meaning            |
|---------|--------------------||
| 90–100  | Definite duplicate |
| 70–89   | Likely duplicate   |
| 40–69   | Related            |
| < 40    | Safe to create     |

If score \< 40:

System recommends creating a new bug.

User clicks **Save Bug**.

Bug is inserted into MongoDB.

------------------------------------------------------------------------

# 4. MongoDB Data Model

Collection: `bugs`

Example:

    {
     "_id": { "$oid": "69b67ccec0173257fb08ab74" },

     "Bug ID": "BUG-0001",

     "Bug Title": "Funds transfer not updating balance #0",

     "Module": "ATM Services",

     "Module Wise": "ATM Services Wise",

     "Steps": "Step 1: Insert card into ATM -> Step 2: Enter PIN -> Step 3: Select option -> Step 4: Confirm -> Step 5: Verify",

     "Expected Results": "ATM PIN change should be processed successfully and the new PIN should work immediately",

     "Actual Results": "System failed to perform as expected",

     "Acceptance Criteria": "Operation must meet functional and business requirements",

     "text": "",

     "embedding": [ -0.0409, 0.0267, 0.0330, ... ]
    }

Field descriptions:

| Field               | Type       | Description                                                    |
|---------------------|------------|----------------------------------------------------------------|
| `_id`               | ObjectId   | MongoDB auto-generated unique identifier                       |
| `Bug ID`            | String     | Human-readable ID, zero-padded format (`BUG-0001`, `BUG-0002`)|
| `Bug Title`         | String     | Short summary of the bug                                       |
| `Module`            | String     | Application module where the bug occurs                        |
| `Module Wise`       | String     | Module categorization/grouping                                 |
| `Steps`             | String     | Reproduction steps, arrow-separated (`->`)                     |
| `Expected Results`  | String     | What should happen                                             |
| `Actual Results`    | String     | What actually happened                                         |
| `Acceptance Criteria`| String    | Conditions the fix must satisfy                                |
| `text`              | String     | Precomputed search text for BM25 indexing                      |
| `embedding`         | Array[1024]| Vector embedding generated from bug content                    |

------------------------------------------------------------------------

# 5. Embedding Strategy

Embedding text formula:

    text = Bug Title + " " + Module + " " + Steps + " " + Expected Results + " " + Actual Results

The computed text is stored in the `text` field for BM25 keyword search.
The embedding vector is generated from this text and stored in the `embedding` field.

Embedding model:

`mistral-embed`

Vector dimensions: **1024**

Vector stored in:

`embedding` field in MongoDB (Array of 1024 float values).

Index configuration files:

-   `src/config/bankingbugs-bm25-index.json` — BM25 text index on the `text` field
-   `src/config/bankingbugs-vector-index.json` — Vector search index on the `embedding` field

------------------------------------------------------------------------

# 6. Logical Architecture

Layers:

**API Layer**

-   Express routes
-   Request validation
-   Payload size limits
-   API versioning (`/v1/`)

**Service Layer**

-   BugService — Bug CRUD operations
-   SearchService — BM25, vector, and hybrid search orchestration
-   DuplicateDetectionService — Similarity scoring and recommendation logic
-   EmbeddingService — Vector embedding generation via AI model
-   LLMService — Re-ranking and similarity summarization
-   LoggingService — Structured request/component logging

**Repository Layer**

-   MongoDB access (CRUD, aggregation, vector queries)

**Infrastructure**

-   MongoDB Atlas
-   Embedding API
-   LLM service (Groq / OpenAI / Mistral)

------------------------------------------------------------------------

# 7. Hybrid Search Pipeline

1.  Receive search query
2.  Generate embedding
3.  Run BM25 keyword search
4.  Run vector similarity search
5.  Merge results
6.  Remove duplicates
7.  LLM re-ranking
8.  Return ranked bugs

------------------------------------------------------------------------

# 8. Similar Bug Search API

POST /v1/bugs/similar

Request:

    {
     "query": "transfer not updating balance",
     "topK": 10,
     "offset": 0
    }

Response:

    {
     "results": [
      {
       "bugId": "BUG-0001",
       "title": "Funds transfer not updating balance #0",
       "similarityScore": 92,
       "confidence": "High"
      }
     ],
     "total": 25,
     "offset": 0,
     "limit": 10
    }

------------------------------------------------------------------------

# 9. Bug Details API

GET /v1/bugs/{bugId}

Returns full bug information.

------------------------------------------------------------------------

# 10. Create Bug API

POST /v1/bugs

Flow:

1.  Generate `text` field (concatenation of Bug Title + Module + Steps + Expected Results + Actual Results)
2.  Generate `embedding` vector from the `text` field
3.  Generate Bug ID (auto-increment, zero-padded: `BUG-0001`, `BUG-0002`, ...)
4.  Derive `Module Wise` from the Module field
5.  Save bug to MongoDB

Bug ID Generation:

A `counters` collection in MongoDB maintains the next Bug ID sequence.
Each new bug increments the counter and receives a zero-padded ID like `BUG-0001`, `BUG-0002`, etc.

Response:

    {
     "message": "Bug created successfully",
     "bugId": "BUG-0042"
    }

------------------------------------------------------------------------

# 11. Duplicate Check API

POST /v1/bugs/check-duplicate

Request:

    {
     "title": "ATM transfer not updating balance",
     "steps": "Insert card enter pin transfer"
    }

Response:

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

Confidence levels:

| Score   | Confidence |
|---------|------------|
| 90–100  | High       |
| 70–89   | Medium     |
| 40–69   | Low        |
| < 40    | None       |

If similarity \< 40:

Recommendation:

`create_new_bug`

------------------------------------------------------------------------

# 12. API Endpoints

GET /v1/health

GET /v1/health/db

POST /v1/bugs/similar

GET /v1/bugs/{bugId}

POST /v1/bugs

POST /v1/bugs/check-duplicate

POST /v1/search/vector

POST /v1/search/bm25

POST /v1/search/hybrid

POST /v1/search/rerank

## Rerank Endpoint Detail

POST /v1/search/rerank

Sends the query bug and candidate bugs to the LLM for semantic re-ranking.

Request:

    {
     "queryBug": {
      "title": "OTP verification fails during withdrawal",
      "module": "ATM Withdrawal",
      "steps": "Insert card -> enter PIN -> select withdrawal -> enter OTP"
     },
     "candidates": [
      {
       "bugId": "BUG-101",
       "title": "Withdrawal fails after OTP verification",
       "similarityScore": 78
      },
      {
       "bugId": "BUG-205",
       "title": "ATM timeout during cash dispensing",
       "similarityScore": 65
      }
     ]
    }

Response:

    {
     "reranked": [
      {
       "bugId": "BUG-101",
       "finalScore": 92,
       "reasoning": "Same module, identical OTP failure path during withdrawal"
      },
      {
       "bugId": "BUG-205",
       "finalScore": 48,
       "reasoning": "Same module but different failure mechanism (timeout vs OTP)"
      }
     ]
    }

## API Versioning

All endpoints use the `/v1/` prefix.
When breaking changes are introduced, a new version prefix (`/v2/`) will be added.
Previous versions will be maintained for a deprecation period of 6 months.

------------------------------------------------------------------------

# 13. Logging

Structured JSON logging with component-level timing breakdown.

Example:

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

All logs include:

-   `requestId` — Unique request identifier for tracing
-   `endpoint` — API route called
-   `durationMs` — Total request duration
-   `statusCode` — HTTP response code
-   `componentTimings` — Per-component latency breakdown for performance debugging

------------------------------------------------------------------------

# 14. Error Handling

Fallback strategy with per-component failure handling:

1.  **LLM re-ranking fails** → Return merged BM25 + vector results without re-ranking
2.  **Vector search fails** → Fall back to BM25 keyword search only
3.  **BM25 fails** → Fall back to vector search only
4.  **Embedding generation fails** → Return BM25 keyword search only (vector search not possible)
5.  **All search methods fail** → Return HTTP 503 with error message

Fallback priority order:

1.  Hybrid search (BM25 + vector + LLM re-ranking)
2.  BM25 + vector (no re-ranking)
3.  BM25 only
4.  Vector only

All failures are logged with component name, error details, and fallback action taken.

------------------------------------------------------------------------

# 15. Security

## Authentication

-   All API endpoints require an API key passed via `x-api-key` header
-   Future: JWT-based authentication for user-level access control

## Input Validation & Sanitization

-   All user inputs (title, steps, query) are sanitized to prevent NoSQL injection
-   Request body validated against JSON schemas using express-validator or Joi
-   Maximum payload size: **1MB** enforced at Express middleware level

## Rate Limiting

-   Rate limiting applied to all endpoints to protect LLM and embedding APIs from abuse
-   Default: 100 requests per minute per API key
-   LLM/embedding endpoints: 30 requests per minute per API key

## Data Security

-   MongoDB connections use TLS encryption
-   Sensitive configuration (API keys, connection strings) stored in environment variables, never in code

------------------------------------------------------------------------

# 16. Future Enhancements

-   AI root cause detection
-   Bug clustering
-   Screenshot similarity detection
-   Test case generation
-   GraphRAG bug knowledge graph
