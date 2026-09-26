# Multi-Document Financial Q&A — Final System Architecture Plan

# 1. Requirement, Scope & Summary

### Problem Statement
Build an enterprise-oriented multi-document financial Q&A platform for investment analysts.

An analyst can upload financial documents for multiple companies, including:
- Earnings reports
- Annual filings
- Investor presentations/decks
- Financial models
- Other supporting documents

Initial supported formats:
- PDF
- DOCX
- XLSX
- TXT

The analyst should be able to ask:
- Direct document questions: “What was Company A's FY2025 revenue?”
- Cross-document/company questions: “Which company had the highest revenue growth?”
- Hybrid questions: “Why did Company A's margins decline?”
- Multi-turn follow-ups: “What about margins?” after a previous comparison.

Answers should be fast, financially accurate, and include source-level citations.

### Functional Requirements
1. Upload and process multiple documents across multiple companies.
2. Extract text, tables, and structured financial information.
3. Support scanned PDFs through OCR fallback.
4. Normalize heterogeneous formats into a common internal representation.
5. Extract key financial metrics during ingestion.
6. Support semantic document retrieval.
7. Support structured financial queries and calculations.
8. Support cross-company and cross-document comparisons.
9. Keep numerical calculations outside the LLM.
10. Generate answers with document-level citations by default.
11. Support more granular citations on follow-up when required.
12. Maintain conversation context across multi-turn questions.
13. Avoid resending the entire conversation to the LLM.
14. Gracefully handle partial/poor extraction and alert the analyst when quality is inadequate.
15. Make ingestion extensible so additional file formats can be added later without redesigning downstream stages.

### Non-Functional Requirements
- Low latency for simple/direct questions.
- Deterministic and accurate numerical calculations.
- Scalable to many companies, documents, and concurrent analysts.
- Traceable answers through provenance.
- Cost-efficient LLM usage.
- Graceful degradation for partially readable documents.
- Modular architecture allowing new parsers, extraction services, and retrieval strategies.
- Conversation isolation: session context belongs only to its conversation.

### Scope
**In scope**
- File ingestion and ETL
- Text/table/financial metric extraction
- OCR fallback
- Normalization
- Context-aware chunking
- Vector retrieval
- Structured financial data storage
- Query classification/planning
- Backend calculations and aggregations
- LLM-based answer synthesis
- Citations/provenance
- Conversation/session context
- Extensible format adapter architecture

**Out of scope**
- Building proprietary OCR/chart understanding from scratch
- Investment recommendations or portfolio decisions
- Autonomous web research
- Full document authoring/editing
- Detailed implementation of individual third-party parsing libraries

---

# 2. High-Level Design

The system is split into five logical stages:

```text
1. Ingestion & ETL
        |
        v
2. Document Store & Context-Aware Chunking
        |
        v
3. Retrieval & Synthesis
        |
        v
4. Session State & Context
        |
        v
5. Answer + Citations
```

A central principle is that the LLM is used for interpretation and synthesis, while deterministic financial computation is performed by backend services.

### Core flow

```text
                     Analyst
                        |
                 Upload / Ask
                        |
              +---------+---------+
              |                   |
              v                   v
        Ingestion API          Query API
              |                   |
              v                   v
        ETL / Normalize     Session Context
              |                   |
       +------+-------+            v
       |      |       |       Query Planner
       v      v       v            |
   Object   Vector  Financial      +----------------+
   Store     DB      Data DB       |       |        |
                                   v       v        v
                               Semantic Structured Hybrid
                               Retrieval Analytics Retrieval
                                   |       |        |
                                   +-------+--------+
                                           |
                                    Results + Evidence
                                           |
                                           v
                                      LLM Synthesis
                                           |
                                           v
                                   Answer + Citations
```

---

# 3. Overall Architecture

## 3.1 Main Components

```text
┌────────────────────────────────────────────────────────────┐
│                         Client / UI                        │
│       Upload Documents | Select Companies | Chat Q&A       │
└─────────────────────────────┬──────────────────────────────┘
                              |
                              v
┌────────────────────────────────────────────────────────────┐
│                     API / Application Layer                │
│ Upload API | Query API | Session API | Auth / Access       │
└───────────────┬──────────────────────────────┬─────────────┘
                |                              |
                v                              v
┌──────────────────────────┐       ┌─────────────────────────┐
│    Ingestion Pipeline    │       │   Query Orchestration   │
│                          │       │                         │
│ Format Adapter Registry  │       │ Session Context Manager │
│ Extractors               │       │ Query Classifier        │
│ OCR fallback             │       │ Query Planner           │
│ Normalizer               │       │ Retrieval Coordinator   │
│ Financial Metric Extract │       │ Synthesis               │
└──────────────┬───────────┘       └────────────┬────────────┘
               |                                |
               v                                v
      ┌────────────────┐             ┌─────────────────────┐
      │ Object Store   │             │ Vector Store        │
      │ Raw Documents  │             │ Embeddings/Chunks   │
      │ Normalized Doc │             └─────────────────────┘
      └────────────────┘
               |
               v
      ┌──────────────────────┐
      │ Metadata / Financial  │
      │ Data Store            │
      │ Metrics + Provenance  │
      └──────────────────────┘
               |
               +----------------------------+
                                            |
                                            v
                                  ┌────────────────────┐
                                  │ Financial Analytics│
                                  │ / Calculation      │
                                  │ Service            │
                                  └────────────────────┘
                                            |
                                            v
                                  ┌────────────────────┐
                                  │ LLM / Synthesis    │
                                  │ Layer              │
                                  └────────────────────┘
                                            |
                                            v
                                      Final Answer
```

## 3.2 Storage Responsibilities

### Object / Document Store
Stores:
- Original uploaded files
- Normalized document representation
- Large document artifacts
- Potentially extracted images/tables

Access pattern:
- Durable document/artifact retrieval
- Rarely accessed during every simple Q&A query

### Vector Store
Stores:
- Embeddings
- Context-aware retrieval chunks
- Relevant metadata for filtering

Access pattern:
- Semantic search during Q&A

### Metadata / Financial Data Store
Stores:
- Company/document metadata
- Reporting periods and document types
- Structured financial metrics
- Provenance references
- Extraction quality/status
- Session-related references where required

Access pattern:
- Fast filters/lookups
- Structured financial queries
- Cross-company comparisons

---

# 4. Stage-Wise Details

## Stage 1 — Format Ingestion Pipeline

### 1.1 File Intake

```text
Upload
  |
  v
File Validation
  |
  +--> MIME/type detection
  +--> Size/security checks
  +--> Duplicate/idempotency checks
  |
  v
Format Adapter Registry
```

### 1.2 Extensible Format Adapter Architecture

Do not hard-code downstream processing around PDF/DOCX/XLSX/TXT.

Use an adapter/plug-in model:

```text
                 Format Adapter Registry
                          |
        +---------+-------+-------+---------+
        |         |       |       |         |
       PDF      DOCX     XLSX    TXT      Future...
        |         |       |       |         |
     Extractor Extractor ...   Extractor  Adapter
        |         |       |       |         |
        +---------+-------+-------+---------+
                          |
                          v
                Common Extracted Model
```

Each adapter should expose a common contract such as:

```text
parse(file)
    -> extracted document
    -> sections
    -> tables
    -> figures/assets
    -> source locations
    -> extraction quality
```

Adding a new format should therefore primarily require:
1. A new format adapter/parser.
2. Mapping its output to the existing normalized model.
3. Format-specific validation/tests.

Downstream chunking, storage, indexing, retrieval, and Q&A should not need to change.

This makes future additions such as CSV, HTML, PPTX, or other financial data formats an incremental change.

### 1.3 Extraction Strategy

PDF:
```text
PDF text/table extractor
          |
     Extraction quality check
          |
      +---+---+
      |       |
   Good      Poor
      |       |
    Use     OCR fallback
```

DOCX:
- Extract paragraphs
- Tables
- Headings/sections
- Relevant embedded assets

XLSX:
- Extract sheets, ranges, labels, values and financial structures
- Prefer actual/computed values
- Preserve useful high-level formula information
- Preserve sheet/range provenance

TXT:
- Direct text ingestion

Complex figures/charts:
- Use a specialized external service or future in-house capability where required.
- Detailed chart-understanding implementation is outside current scope.

### 1.4 Normalization

All adapters map into a common internal model:

```text
NormalizedDocument
├── document_metadata
├── company_context
├── reporting_context
├── sections
├── tables
├── figures/assets
├── financial_metrics
├── provenance
└── extraction_quality
```

Example provenance:

```text
PDF:
company = Microsoft
document = FY2025 Annual Report
section = Consolidated Income Statement
page = 42

XLSX:
company = Microsoft
document = Financial Model.xlsx
sheet = Income Statement
range = B5:F12
```

### 1.5 Financial Metric Extraction

During ETL, identify common financial concepts such as:
- Revenue
- Gross profit
- Operating income
- Net income
- Margins
- EPS
- Growth
- Cash flow
- Other relevant metrics

Store them in structured form with:
- company
- metric
- period
- value
- unit/currency
- source document
- source location
- confidence/status

The goal is to make common financial questions cheap and deterministic.

---

## Stage 2 — Document Store & Context-Aware Chunking

### 2.1 Context-Aware Chunking

Do not use one generic token-size chunk for every document.

Examples:

```text
Management Discussion
   -> semantic/section-aware text chunks

Income Statement
   -> logical financial table units

XLSX financial model
   -> sheet/range/metric-aware units
```

Chunking should preserve:
- Company
- Document
- Reporting context
- Section
- Financial statement/table identity
- Time period
- Source location

### 2.2 Embeddings

Generate embeddings for retrieval-ready content.

Typical flow:

```text
Normalized Content
      |
      v
Context-Aware Chunks
      |
      v
Embedding Model
      |
      v
Vector DB
```

The vector DB is for retrieval; structured financial data remains the source for deterministic financial computation.

---

## Stage 3 — Retrieval & Synthesis Engine

### 3.1 Query Classification

Every query first goes through a query classifier/planner.

```text
User Question
      |
      v
Query Classifier
      |
 +----+--------+
 |             |
 v             v
Direct      Requires work
answer      / computation
 |             |
 v             +---------+
Cheap path               |
                          v
                     Structured / Hybrid
```

Three useful classes:

**Semantic**
> “What did Company A say about AI investment?”

**Structured**
> “Which company had the highest revenue growth?”

**Hybrid**
> “Why did Company's margin decline?”

### 3.2 Direct/Semantic Path

```text
Question
   |
   v
Context Resolution
   |
   v
Vector Search
   |
   v
Relevant Chunks
   |
   v
LLM Synthesis
   |
   v
Answer + document citations
```

Optimize this path for low latency and low token usage.

### 3.3 Structured Financial Path

Example:

> “Which company had the highest revenue growth?”

```text
Question
   |
   v
Query Planner
   |
   v
Financial Data Query
   |
   v
Revenue FY2024 + FY2025
   |
   v
Calculation Service
   |
   v
Growth by Company
   |
   v
Rank/Select Result
   |
   v
Retrieve supporting source references
   |
   v
LLM Synthesis
```

The LLM should not perform the actual arithmetic.

### 3.4 Hybrid Path

Example:

> “Why did Company A's margins decline?”

```text
Question
   |
   v
Query Planner
   |
   +------------------+
   |                  |
   v                  v
Financial Data       Vector Search
   |                  |
   |                  +--> Explanatory evidence
   v
Calculate margin     |
change               |
   |                  |
   +--------+---------+
            |
            v
       LLM Synthesis
            |
            v
      Answer + citations
```

### 3.5 Calculation Service

All calculations, aggregations, comparisons, rankings, and transformations should occur outside the LLM.

Examples:
- Revenue growth
- CAGR
- Margin change
- Cross-company ranking
- Period comparisons
- Percentage changes
- Aggregations across documents

This provides:
- Better numerical accuracy
- Lower hallucination risk
- Lower token usage
- More predictable cost
- Reproducibility

---

## Stage 4 — Session State & Context

### 4.1 Conversation-Scoped Context

Context is strictly conversation-scoped.

```text
Conversation A
   -> Context A

Conversation B
   -> Context B
```

No automatic carry-over between unrelated conversations.

### 4.2 Structured Session State

Instead of passing the entire previous conversation to the LLM, maintain a compact structured session state and a bounded conversation window.

```text
Session State
├── companies
├── selected documents
├── reporting periods
├── current task/query intent
├── metrics under discussion
├── filters
├── relevant previous results
└── Entity Registry
```

Example:

```text
companies = [Apple, Microsoft, Google]
period = FY2025
task = comparison
metric = revenue_growth
```

Then:

> “What about margins?”

can resolve to:

```text
companies = [Apple, Microsoft, Google]
period = FY2025
metric = margins
```

Only relevant context is forwarded, reducing:
- Latency
- Token cost
- Context noise
- Hallucination opportunities

### 4.3 Sliding Window + Explicit Truncation

Use a **sliding window** for raw conversational context rather than continuously growing the prompt.

Example policy:

```text
Maximum conversational window:
- Last 10 turns OR
- 4,000 words
- whichever limit is reached first
```

When a new turn would push the conversation past either limit:

```text
New user turn
      |
      v
Append turn to window
      |
      v
Limit exceeded?
   +--+--+
   |     |
  No    Yes
   |     |
   v     v
Keep   Drop oldest turns
        until window <= limit
```

The truncation rule is explicit and deterministic. The newest turns remain available, while the oldest raw conversation turns are discarded first.

The system must not rely on dropped conversation history to resolve follow-up questions. Context that needs to survive truncation is maintained separately in the Entity Registry.

### 4.4 Entity Registry

Maintain a **separate structured store** for durable session context.

```text
Entity Registry
├── Companies mentioned
├── Documents in scope
├── Reporting periods in scope
├── Current task / comparison
├── Most recent metric
├── Most recent topic
├── Active filters
└── Other explicitly resolved entities
```

Example:

```text
companies = [Apple, Microsoft, Google]
documents = [FY2025 Annual Reports]
period = FY2025
current_task = cross-company comparison
most_recent_metric = revenue_growth
most_recent_topic = financial performance
```

When older turns have been dropped from the sliding window, follow-up questions resolve against this registry instead of re-reading the discarded conversation.

For example:

```text
Raw window:
  Turn 8: "Compare Apple, Microsoft and Google revenue growth."
  Turn 9: "Which one is highest?"
  Turn 10: "What about margins?"

Entity Registry:
  companies = [Apple, Microsoft, Google]
  period = FY2025
  metric/topic = margins
```

This provides persistent conversational context without retaining the full transcript.

### 4.5 Why Summarization Is Not Used

**Conversation summarization is explicitly not used for session continuity.**

The reason is accuracy and verifiability: a summary can silently alter, omit, or compress important financial data, figures, qualifiers, document references, or source context. A user may then be unable to verify whether a later answer is based on the original statement or on an imperfect summary.

For a financial Q&A system where figures and citations need to remain trustworthy, deterministic truncation plus a structured Entity Registry is preferred over generated summaries.

### 4.6 Ambiguity

When a follow-up cannot be confidently resolved from the sliding window and Entity Registry:

```text
User Question
      |
Context Resolution
      |
 +----+----+
 |         |
Clear    Ambiguous
 |         |
 v         v
Proceed  Ask clarification
```

This is preferable to silently assuming the wrong financial context.

---

## Stage 5 — Answer & Citations

### Default citation strategy

Keep normal responses lightweight:

```text
Answer
  |
  +-- Document-level citations
```

Example:

> Company A had the highest FY2025 revenue growth.
> Sources: FY2025 Annual Report; FY2025 Earnings Report.

### Granular citations on demand

When the analyst asks for more detail:

```text
Document citation
      |
      v
Resolve provenance
      |
      v
Page / section / table / sheet / range
```

Examples:
- PDF → page + section
- DOCX → section
- XLSX → sheet + range/cell

This avoids doing expensive provenance resolution for every basic response.

---

# 5. Other Information

## 5.1 Recommended Service Boundaries

A practical service decomposition could be:

```text
API Gateway
   |
   +-- Document Service
   +-- Ingestion/ETL Service
   +-- Chunking/Indexing Service
   +-- Query Orchestrator
   +-- Retrieval Service
   +-- Financial Analytics Service
   +-- Session Context Service
   +-- LLM/Synthesis Service
```

For an initial implementation, some of these can be deployed together as a modular service rather than separate microservices. Split them when scale or team boundaries justify it.

## 5.2 Asynchronous Ingestion

Document processing should generally be asynchronous:

```text
Upload
  |
  v
Store file
  |
  v
Create ingestion job
  |
  v
Queue
  |
  v
Worker
  |
  +--> Extract
  +--> Normalize
  +--> Financial metric extraction
  +--> Chunk
  +--> Embed
  +--> Index
  |
  v
Ready
```

This prevents large PDFs/spreadsheets from blocking the upload/API request.

## 5.3 Ingestion Status

Expose a simple state machine:

```text
UPLOADED
  -> PROCESSING
  -> READY

PROCESSING
  -> PARTIAL
  -> FAILED
```

`PARTIAL` can be used when useful content was extracted but some sections/assets had problems.

## 5.4 Observability

Track:
- Extraction failures
- OCR fallback rate
- Extraction quality
- Chunk/indexing failures
- Query classification
- Retrieval latency
- Calculation latency
- LLM latency
- Token usage
- Citation generation success
- End-to-end answer latency

This will make it easier to identify whether problems originate from ingestion, retrieval, calculation, context resolution, or synthesis.

## 5.5 Security / Enterprise Considerations

Because financial documents may be sensitive:
- Authenticate users
- Authorize access to documents and conversations
- Keep tenant/company boundaries explicit if multi-tenant
- Encrypt data in transit and at rest
- Avoid sending unnecessary document content to external LLM/OCR services
- Log access and major processing events
- Keep conversation context isolated

## 5.6 Key Design Principle

The overall architecture deliberately separates four responsibilities:

```text
Understand documents
        ↓
Retrieve evidence
        ↓
Calculate financial facts
        ↓
Explain results
```

The system should rely on the **document/structured data layer for facts**, the **analytics service for numbers**, and the **LLM for interpretation and natural-language synthesis**.

This separation is the main mechanism for achieving a fast, scalable, cost-efficient, and financially reliable Q&A experience.
