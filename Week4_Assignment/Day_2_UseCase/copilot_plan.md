# Copilot Plan – AI Duplicate Bug Detection System

> **NOTE:** This is a build plan only. No code is generated.
> Follow the order: **API-first, DB-second.**

------------------------------------------------------------------------

## Build Order Overview

    Phase 1: API Layer (Routes + Middleware + Validation)
    Phase 2: Database Layer (MongoDB connection + Repository + Indexes)
    Phase 3: Service Layer (Business logic)
    Phase 4: Integration (Wire services to routes)
    Phase 5: Seed Data & Testing

------------------------------------------------------------------------

## Phase 1: API Layer (Build First)

### Step 1.1 — Project Setup

- Initialize Node.js project with TypeScript
- Create `tsconfig.json`
- Install dependencies: `express`, `typescript`, `dotenv`, `uuid`, `express-validator`, `express-rate-limit`
- Create folder structure:

```
src/
  app.ts
  server.ts
  routes/
  middleware/
  services/
  repositories/
  config/
  data/
  types/
```

### Step 1.2 — Create Type Definitions

**File:** `src/types/bug.types.ts`

- `Bug` interface — All fields from MongoDB document (`Bug ID`, `Bug Title`, `Module`, `Module Wise`, `Steps`, `Expected Results`, `Actual Results`, `Acceptance Criteria`, `text`, `embedding`)
- `BugCreateRequest` interface — Fields required to create a bug (title, module, steps, expected, actual, acceptance criteria)
- `BugSearchRequest` interface — `query`, `topK`, `offset`
- `DuplicateCheckRequest` interface — `title`, `steps`
- `SearchResult` interface — `bugId`, `title`, `similarityScore`, `confidence`
- `DuplicateCheckResponse` interface — `duplicates[]`, `possibleDuplicates[]`, `recommendation`
- `RerankRequest` interface — `queryBug`, `candidates[]`
- `RerankResponse` interface — `reranked[]` with `bugId`, `finalScore`, `reasoning`

### Step 1.3 — Create Middleware

**File:** `src/middleware/requestId.ts`

- Generate UUID for each request, attach to `req` and response header

**File:** `src/middleware/auth.ts`

- Read `x-api-key` header
- Compare against `API_KEY` env variable
- Skip auth for health endpoints
- Return 401 if invalid

**File:** `src/middleware/rateLimiter.ts`

- Default limiter: 100 requests/min per API key
- AI limiter: 30 requests/min for `/v1/search/rerank`, embedding endpoints

**File:** `src/middleware/validator.ts`

- Validation schemas for each endpoint using express-validator
- `validateBugSearch` — query required (min 3 chars), topK optional (1-50), offset optional
- `validateBugCreate` — Bug Title required, Module required, Steps required, Expected Results required, Actual Results required
- `validateDuplicateCheck` — title required, steps required
- `validateRerank` — queryBug object required, candidates array required

**File:** `src/middleware/errorHandler.ts`

- Catch-all error middleware
- Return structured JSON: `{ error, details, requestId }`
- Log error with requestId

### Step 1.4 — Create Route Files (Stub Handlers)

**File:** `src/routes/health.routes.ts`

- `GET /` — Return `{ status: "ok" }`
- `GET /db` — Call DB health check (stub: return placeholder)

**File:** `src/routes/bug.routes.ts`

- `POST /similar` — Validate input, call SearchService.hybridSearch (stub)
- `GET /:bugId` — Call BugService.getBugById (stub)
- `POST /` — Validate input, call BugService.createBug (stub)
- `POST /check-duplicate` — Validate input, call DuplicateDetectionService.check (stub)

**File:** `src/routes/search.routes.ts`

- `POST /bm25` — Call SearchService.bm25Search (stub)
- `POST /vector` — Call SearchService.vectorSearch (stub)
- `POST /hybrid` — Call SearchService.hybridSearch (stub)
- `POST /rerank` — Validate input, call LLMService.rerank (stub)

### Step 1.5 — Create Express App

**File:** `src/app.ts`

- Create Express app
- Register middleware in order: body parser (1MB limit) → requestId → auth → rateLimiter
- Register routes: `/v1/health`, `/v1/bugs`, `/v1/search`
- Register errorHandler last

**File:** `src/server.ts`

- Load env variables via dotenv
- Import app
- Start listening on configured PORT
- Log startup message

------------------------------------------------------------------------

## Phase 2: Database Layer (Build Second)

### Step 2.1 — MongoDB Connection

**File:** `src/config/database.ts`

- `connectToDatabase()` — Connect to MongoDB Atlas using `MONGODB_URI` env variable
- `getDatabase()` — Return database instance
- `closeDatabaseConnection()` — Graceful disconnect
- Connection error handling and retry logic

### Step 2.2 — Bug Repository

**File:** `src/repositories/bug.repository.ts`

**Class:** `BugRepository`

Methods:

- `findByBugId(bugId: string)` — `findOne({ "Bug ID": bugId })`
- `insertBug(bug: Bug)` — `insertOne()` into bugs collection
- `getNextBugId()` — `findOneAndUpdate()` on counters collection, increment sequence, return zero-padded ID (`BUG-0001`)
- `bm25Search(queryText: string, topK: number)` — MongoDB `$search` aggregation pipeline using Atlas Search index
- `vectorSearch(embedding: number[], topK: number)` — MongoDB `$vectorSearch` aggregation pipeline using Atlas Vector Search index
- `healthCheck()` — Ping MongoDB, return connection status

### Step 2.3 — Create Indexes

Use existing config files:

- `src/config/bankingbugs-bm25-index.json` — Apply as Atlas Search index on `bugs` collection
- `src/config/bankingbugs-vector-index.json` — Apply as Atlas Vector Search index on `bugs` collection

### Step 2.4 — Seed Data Loader

**File:** `src/config/seedData.ts`

- Read `src/data/banking_bugs.csv`
- Parse CSV rows into bug documents
- For each bug: generate `text` field, generate `embedding` via EmbeddingService
- Insert into `bugs` collection
- Initialize `counters` collection with starting sequence value

------------------------------------------------------------------------

## Phase 3: Service Layer

### Step 3.1 — Embedding Service

**File:** `src/services/embedding.service.ts`

**Class:** `EmbeddingService`

Methods:

- `generateEmbedding(text: string): Promise<number[]>` — Call Mistral embed API, return 1024-dim vector
- `generateSearchText(bug: BugCreateRequest): string` — Concatenate: Bug Title + " " + Module + " " + Steps + " " + Expected Results + " " + Actual Results

### Step 3.2 — Search Service

**File:** `src/services/search.service.ts`

**Class:** `SearchService`

Dependencies: `BugRepository`, `EmbeddingService`

Methods:

- `bm25Search(query: string, topK: number)` — Call repository BM25 search
- `vectorSearch(query: string, topK: number)` — Generate embedding, call repository vector search
- `hybridSearch(query: string, topK: number)` — Run BM25 + vector in parallel, merge results, deduplicate by Bug ID, sort by score
- `mergeResults(bm25Results, vectorResults)` — Combine and deduplicate, normalize scores

### Step 3.3 — LLM Service

**File:** `src/services/llm.service.ts`

**Class:** `LLMService`

Methods:

- `rerankBugs(queryBug, candidates[])` — Send query + candidates to LLM, parse ranked response with scores and reasoning
- `buildRerankPrompt(queryBug, candidates[])` — Construct LLM prompt for re-ranking task

### Step 3.4 — Duplicate Detection Service

**File:** `src/services/duplicateDetection.service.ts`

**Class:** `DuplicateDetectionService`

Dependencies: `SearchService`, `LLMService`

Methods:

- `checkDuplicate(title, steps)` — Run hybrid search → LLM rerank → classify results into duplicates/possibleDuplicates → generate recommendation
- `classifyResults(results[])` — Apply score thresholds (≥70 = duplicate, 40-69 = possibleDuplicate, <40 = ignore)
- `getRecommendation(duplicates[])` — Return `"duplicate"`, `"possible_duplicate"`, or `"create_new_bug"`
- `getConfidence(score)` — Map score to confidence level (High/Medium/Low/None)

### Step 3.5 — Bug Service

**File:** `src/services/bug.service.ts`

**Class:** `BugService`

Dependencies: `BugRepository`, `EmbeddingService`

Methods:

- `getBugById(bugId: string)` — Fetch from repository, strip embedding from response
- `createBug(request: BugCreateRequest)` — Generate text → generate embedding → get next Bug ID → derive Module Wise → insert → return bugId
- `deriveModuleWise(module: string)` — Append "Wise" to module name

### Step 3.6 — Logging Service

**File:** `src/services/logging.service.ts`

**Class:** `LoggingService`

Methods:

- `logRequest(requestId, endpoint, statusCode, durationMs, componentTimings)` — Output structured JSON log
- `logError(requestId, endpoint, error, fallbackAction)` — Log error with fallback info

------------------------------------------------------------------------

## Phase 4: Integration (Wire Everything Together)

### Step 4.1 — Connect Routes to Services

- Update `health.routes.ts` — Wire `GET /db` to `BugRepository.healthCheck()`
- Update `bug.routes.ts` — Wire handlers to `BugService`, `SearchService`, `DuplicateDetectionService`
- Update `search.routes.ts` — Wire handlers to `SearchService`, `LLMService`

### Step 4.2 — Add Fallback Logic

In `SearchService.hybridSearch()`:

    try full hybrid (BM25 + vector + rerank)
    catch LLM failure → return BM25 + vector merged (no rerank)
    catch vector failure → return BM25 only
    catch BM25 failure → return vector only
    catch all failures → throw 503

### Step 4.3 — Add Component Timing

In each route handler:

- Record start time before each service call
- Capture per-component durations
- Pass timings to LoggingService

------------------------------------------------------------------------

## Phase 5: Seed Data & Testing

### Step 5.1 — Load Seed Data

- Run seed script to load `banking_bugs.csv` into MongoDB
- Generate `text` and `embedding` for each bug
- Verify BM25 and vector indexes are active

### Step 5.2 — Manual API Testing

Test each endpoint in order:

1. `GET /v1/health` — Verify server running
2. `GET /v1/health/db` — Verify MongoDB connected
3. `POST /v1/bugs/similar` — Search and verify ranked results
4. `GET /v1/bugs/BUG-0001` — Verify bug details returned
5. `POST /v1/bugs/check-duplicate` — Verify duplicate detection with scores
6. `POST /v1/bugs` — Create a new bug, verify ID generated
7. `POST /v1/search/bm25` — Verify keyword search
8. `POST /v1/search/vector` — Verify vector search
9. `POST /v1/search/hybrid` — Verify merged results
10. `POST /v1/search/rerank` — Verify LLM re-ranking

------------------------------------------------------------------------

## Component Summary

| Component                     | File                                  | Depends On                    |
|-------------------------------|---------------------------------------|-------------------------------|
| Type Definitions              | `types/bug.types.ts`                  | —                             |
| Request ID Middleware         | `middleware/requestId.ts`             | `uuid`                        |
| Auth Middleware               | `middleware/auth.ts`                  | env `API_KEY`                 |
| Rate Limiter Middleware       | `middleware/rateLimiter.ts`           | `express-rate-limit`          |
| Validator Middleware          | `middleware/validator.ts`             | `express-validator`           |
| Error Handler Middleware      | `middleware/errorHandler.ts`          | LoggingService                |
| Health Routes                 | `routes/health.routes.ts`            | BugRepository                 |
| Bug Routes                    | `routes/bug.routes.ts`               | BugService, SearchService, DuplicateDetectionService |
| Search Routes                 | `routes/search.routes.ts`            | SearchService, LLMService     |
| Database Config               | `config/database.ts`                 | `mongodb`, env `MONGODB_URI`  |
| Bug Repository                | `repositories/bug.repository.ts`     | Database Config               |
| Embedding Service             | `services/embedding.service.ts`      | env `MISTRAL_API_KEY`         |
| Search Service                | `services/search.service.ts`         | BugRepository, EmbeddingService |
| LLM Service                   | `services/llm.service.ts`            | env `LLM_API_KEY`             |
| Duplicate Detection Service   | `services/duplicateDetection.service.ts` | SearchService, LLMService |
| Bug Service                   | `services/bug.service.ts`            | BugRepository, EmbeddingService |
| Logging Service               | `services/logging.service.ts`        | —                             |
| Seed Data Loader              | `config/seedData.ts`                 | BugRepository, EmbeddingService |
| Express App                   | `app.ts`                             | All middleware, all routes     |
| Server Entry                  | `server.ts`                          | App, Database Config           |

------------------------------------------------------------------------

## Build Sequence (Dependency Order)

    1.  Project setup + tsconfig + install packages
    2.  types/bug.types.ts
    3.  middleware/requestId.ts
    4.  middleware/auth.ts
    5.  middleware/rateLimiter.ts
    6.  middleware/validator.ts
    7.  middleware/errorHandler.ts
    8.  routes/health.routes.ts (stubs)
    9.  routes/bug.routes.ts (stubs)
    10. routes/search.routes.ts (stubs)
    11. app.ts (wire middleware + routes)
    12. server.ts (start server)
    ── API layer complete, server starts with stub responses ──
    13. config/database.ts
    14. repositories/bug.repository.ts
    ── DB layer complete ──
    15. services/embedding.service.ts
    16. services/logging.service.ts
    17. services/search.service.ts
    18. services/llm.service.ts
    19. services/duplicateDetection.service.ts
    20. services/bug.service.ts
    ── Service layer complete ──
    21. Wire routes to real services
    22. Add fallback logic
    23. Add component timing
    ── Integration complete ──
    24. config/seedData.ts
    25. Load seed data + test all endpoints
    ── System ready ──

------------------------------------------------------------------------

*Document Version: 1.0*
*Date: March 15, 2026*
*Project: RAG-based Duplicate Bug Detection System*
