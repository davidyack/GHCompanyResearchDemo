# Data Model — GH Company Research Demo

This document describes the data model of the GH Company Research Demo application — a multi-agent AI platform that researches a company by crawling the web, synthesising findings with LLMs, and streaming a structured report to the user.

---

## Table of Contents

1. [Technology Stack](#technology-stack)
2. [Core State Objects (Python)](#core-state-objects-python)
   - [InputState](#inputstate)
   - [ResearchState](#researchstate)
   - [Document](#document)
   - [Job Status Tracker](#job-status-tracker)
3. [API Layer](#api-layer)
   - [Request Models](#request-models)
   - [Endpoints](#endpoints)
   - [SSE Event Types](#sse-event-types)
4. [MongoDB Persistence](#mongodb-persistence)
   - [jobs Collection](#jobs-collection)
   - [reports Collection](#reports-collection)
5. [Frontend Types (TypeScript)](#frontend-types-typescript)
6. [Data Flow](#data-flow)
7. [Entity Relationship Overview](#entity-relationship-overview)

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI + Uvicorn |
| AI orchestration | LangGraph + LangChain |
| LLM – report compilation | OpenAI GPT-4o (`langchain-openai`) |
| LLM – research briefings | Google Gemini 2.5 Flash (`langchain-google-genai`) |
| Web search & extraction | Tavily (`tavily-python`) |
| Database | MongoDB (`pymongo`) |
| PDF generation | ReportLab |
| Frontend | React 18 + TypeScript, Vite, Tailwind CSS |

---

## Core State Objects (Python)

State is passed through a LangGraph pipeline. Each node reads from and writes to the shared `ResearchState` dict.

### InputState

Defined in `backend/classes/state.py`.

```python
class InputState(TypedDict, total=False):
    company:      Required[str]   # Company name (required)
    company_url:  NotRequired[str] # Company website URL
    hq_location:  NotRequired[str] # Headquarters location
    industry:     NotRequired[str] # Industry classification
    job_id:       NotRequired[str] # UUID that tracks this research job
```

| Field | Type | Required | Description |
|---|---|---|---|
| `company` | `str` | ✅ | Name of the company being researched |
| `company_url` | `str` | ❌ | Homepage URL; used for the initial website crawl |
| `hq_location` | `str` | ❌ | Headquarters location for contextual research |
| `industry` | `str` | ❌ | Industry classification to focus searches |
| `job_id` | `str` | ❌ | UUID assigned at job creation; links state to job tracker and MongoDB |

---

### ResearchState

`ResearchState` extends `InputState` and accumulates all intermediate and final artefacts as documents move through the pipeline.

```python
class ResearchState(InputState):
    # Raw research data — populated by the four parallel Research nodes
    site_scrape:       Dict[str, Any]  # URL → page content from company website
    financial_data:    Dict[str, Any]  # URL → financial document
    news_data:         Dict[str, Any]  # URL → news article document
    industry_data:     Dict[str, Any]  # URL → industry analysis document
    company_data:      Dict[str, Any]  # URL → company overview document

    # Curated data — filtered by Curator (Tavily score ≥ 0.4, max 30 per category)
    curated_financial_data: Dict[str, Any]
    curated_news_data:      Dict[str, Any]
    curated_industry_data:  Dict[str, Any]
    curated_company_data:   Dict[str, Any]

    # Briefings — Gemini 2.5 Flash summaries, one per category
    financial_briefing: str
    news_briefing:      str
    industry_briefing:  str
    company_briefing:   str

    # Final report output
    report:     str           # Compiled markdown report (GPT-4o)
    references: List[str]     # Ordered list of reference URLs
    reference_titles: Dict[str, str]  # URL → page title
    reference_info:   Dict[str, Any]  # URL → metadata
    briefings:        Dict[str, Any]  # All briefings keyed by category

    # Internal tracking
    messages: List[Any]  # LangChain AIMessage log from each node
```

#### Raw & Curated Data Dicts

Each `*_data` and `curated_*_data` field is a `Dict[url: str → Document]`. The URL is the unique key.

After curation:
- Documents with Tavily relevance score `< 0.4` are removed (company-website documents are always kept).
- At most **30** documents are kept per category, sorted by score descending.

After enrichment, curated documents have a `raw_content` field added in-place by the Enricher node.

---

### Document

A single research document as it flows through the pipeline.

| Field | Type | Present after | Description |
|---|---|---|---|
| `url` | `str` | Grounding / Research | Canonical URL (query string and fragment stripped) |
| `title` | `str` | Research | Page or article title |
| `content` | `str` | Research | Short snippet returned by Tavily search |
| `raw_content` | `str` | Enricher | Full page content fetched by Tavily extract |
| `query` | `str` | Research | Search query that surfaced this document |
| `source` | `str` | Research | `"web_search"` or `"company_website"` |
| `score` | `float` | Research | Tavily relevance score (0 – 1) |
| `doc_type` | `str` | Curator | Category: `"company"`, `"financial"`, `"news"`, or `"industry"` |
| `evaluation.overall_score` | `float` | Curator | Copy of `score` stored after evaluation |
| `evaluation.query` | `str` | Curator | Copy of `query` stored after evaluation |

---

### Job Status Tracker

An in-memory `defaultdict` (`job_status`) defined in `backend/classes/state.py` and shared across nodes and the API layer. One entry per active job.

```python
job_status[job_id] = {
    "status":       str,        # "pending" | "processing" | "completed" | "failed"
    "result":       Any,        # Reserved for future use
    "error":        str | None, # Error message on failure
    "debug_info":   list,       # Internal debug log entries
    "company":      str | None, # Company name (set on completion)
    "report":       str | None, # Final markdown report (set on completion)
    "current_step": str,        # Name of the LangGraph node currently executing
    "last_update":  str,        # ISO 8601 timestamp of the last write
    "events":       list        # FIFO queue of SSE event dicts streamed to the UI
}
```

This dict is **ephemeral** — it is not written to MongoDB. On restart it is empty. MongoDB is the durable store.

---

## API Layer

Defined in `application.py`.

### Request Models

#### `ResearchRequest`

Accepted body for `POST /research`.

| Field | Type | Required | Description |
|---|---|---|---|
| `company` | `str` | ✅ | Company name |
| `company_url` | `str \| None` | ❌ | Company website URL |
| `industry` | `str \| None` | ❌ | Industry classification |
| `hq_location` | `str \| None` | ❌ | Headquarters location |

#### `PDFGenerationRequest`

Accepted body for `POST /generate-pdf`.

| Field | Type | Required | Description |
|---|---|---|---|
| `report_content` | `str` | ✅ | Markdown text of the report to render |
| `company_name` | `str \| None` | ❌ | Used to name the output PDF file |

---

### Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Health check |
| `POST` | `/research` | Submit a new research job; returns `job_id` |
| `GET` | `/research/{job_id}` | Retrieve job details from MongoDB |
| `GET` | `/research/{job_id}/stream` | Stream real-time SSE events until completion |
| `GET` | `/research/{job_id}/report` | Retrieve the final report (in-memory or MongoDB) |
| `POST` | `/generate-pdf` | Convert a markdown report to a PDF stream |
| `GET` | `/research/pdf/{filename}` | Download a previously generated PDF file |

**`POST /research` response**

```json
{
  "status": "accepted",
  "job_id": "<uuid>",
  "message": "Research started. Connect to /research/{job_id}/stream for updates."
}
```

---

### SSE Event Types

Events are pushed on `GET /research/{job_id}/stream` as JSON-encoded `data:` lines.

| `type` | Emitted by | Key fields | Description |
|---|---|---|---|
| `progress` | API layer | `step` | LangGraph node transition |
| `research_init` | Grounding | — | Research job initialised |
| `crawl_start` | Grounding | — | Website crawl started |
| `crawl_success` | Grounding | — | Website crawl finished |
| `query_generating` | Research nodes | `category` | Query generation in progress |
| `query_generated` | Research nodes | `category`, `query` | A search query is ready |
| `search_complete` | Research nodes | `category`, `count` | Search results collected |
| `curation` | Curator | `category`, `total` | Documents filtered for a category |
| `enrichment` | Enricher | `category`, `enriched`, `total` | Raw content fetched for a category |
| `briefing_start` | Briefing | `category` | Briefing generation started |
| `briefing_complete` | Briefing | `category` | Briefing generation complete |
| `report_chunk` | Editor | `chunk` | Streaming chunk of the final report |
| `complete` | API layer | `report` | Research finished; full report included |
| `error` | API layer | `error` | Failure message |

---

## MongoDB Persistence

**Database**: `tavily_research`

MongoDB is optional. When `MONGODB_URI` is set the application writes to it; otherwise only the in-memory `job_status` dict is used.

### `jobs` Collection

Created by `MongoDBService.create_job()`, updated by `update_job()`.

| Field | Type | Description |
|---|---|---|
| `_id` | `ObjectId` | MongoDB document identifier |
| `job_id` | `str` | UUID matching the in-memory key |
| `inputs.company` | `str` | Company name |
| `inputs.company_url` | `str` | Website URL |
| `inputs.industry` | `str` | Industry classification |
| `inputs.hq_location` | `str` | Headquarters location |
| `status` | `str` | `"pending"` \| `"processing"` \| `"completed"` \| `"failed"` |
| `result` | `object` | Reserved |
| `error` | `str` | Error message on failure |
| `created_at` | `ISODate` | UTC timestamp when the job was created |
| `updated_at` | `ISODate` | UTC timestamp of the most recent status update |

### `reports` Collection

Created by `MongoDBService.store_report()` when a job completes successfully.

| Field | Type | Description |
|---|---|---|
| `_id` | `ObjectId` | MongoDB document identifier |
| `job_id` | `str` | UUID linking back to the `jobs` collection |
| `report_content` | `str` | Full markdown report text |
| `references` | `[str]` | Ordered list of reference URLs |
| `sections` | `[str]` | Sections completed (from `sections_completed`) |
| `analyst_queries` | `object` | Per-category search queries used |
| `created_at` | `ISODate` | UTC timestamp when the report was stored |

---

## Frontend Types (TypeScript)

Defined in `ui/src/types/index.ts`.

### Core Data Types

```typescript
// Status update shown in the progress panel
type ResearchStatusType = {
  step:    string;  // Human-readable step name
  message: string;  // Detail message
};

// Final research output delivered to the UI
type ResearchOutput = {
  summary: string;
  details: {
    report: string;  // Full markdown report
  };
};

// Document enrichment progress counters per category
type EnrichmentCounts = {
  company:   { total: number; enriched: number };
  industry:  { total: number; enriched: number };
  financial: { total: number; enriched: number };
  news:      { total: number; enriched: number };
};
```

### UI / Component Types

```typescript
// Glassmorphism CSS class sets used for theming
type GlassStyle = {
  base:  string;
  card:  string;
  input: string;
};

// Animation CSS class sets
type AnimationStyle = {
  fadeIn:  string;
  writing: string;
};
```

### Component Prop Types

```typescript
type ResearchStatusProps = {
  status:       ResearchStatusType | null;
  error:        string | null;
  isComplete:   boolean;
  currentPhase: 'search' | 'enrichment' | 'briefing' | 'complete' | null;
  isResetting:  boolean;
  glassStyle:   GlassStyle;
  loaderColor:  string;
  statusRef:    React.RefObject<HTMLDivElement>;
};

type ResearchQueriesProps = {
  queries: Array<{
    text:     string;
    number:   number;
    category: string;
  }>;
  streamingQueries: {
    [key: string]: {
      text:       string;
      number:     number;
      category:   string;
      isComplete: boolean;
    };
  };
  isExpanded:     boolean;
  onToggleExpand: () => void;
  isResetting:    boolean;
  glassStyle:     string;
};
```

---

## Data Flow

The diagram below traces how data is created, transformed, and consumed at each stage of the pipeline.

```
Client
  │
  ├─ POST /research  ──────────────────────────────────────────────────────┐
  │   ResearchRequest {company, company_url, industry, hq_location}        │
  │                                                                         ▼
  │                                                            Create job_id (UUID)
  │                                                            job_status[job_id] = {status: "pending"}
  │                                                            mongodb.create_job()
  │                                                                         │
  │   ◄── {status: "accepted", job_id}                                     ▼
  │                                                            InputState assembled
  │                                                                         │
  ├─ GET /research/{job_id}/stream                                          ▼
  │   (SSE — polls job_status[job_id].events)              ┌──────────────────────────┐
  │                                                         │  Grounding Node           │
  │                                                         │  Tavily crawl company     │
  │                                                         │  URL → site_scrape        │
  │                                                         │  {URL → {raw_content}}    │
  │                                                         └───────────┬──────────────┘
  │                                                                     │
  │                                                   ┌────────────────┼────────────────┐
  │                                                   ▼                ▼                ▼
  │                                          ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │                                          │ CompanyAnalyzer│ │FinancialAnalyst│ │ NewsScanner  │
  │                                          │ company_data  │ │financial_data │ │ news_data    │
  │                                          └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
  │                                                 │                │                │
  │                                          ┌──────┴────────────────┴────────────────┴───────┐
  │                                          │           IndustryAnalyzer                      │
  │                                          │           industry_data                          │
  │                                          └──────────────────────┬──────────────────────────┘
  │                                                                  │
  │                                                                  ▼
  │                                                        ┌──────────────────┐
  │                                                        │ Collector Node    │
  │                                                        │ validates all     │
  │                                                        │ data present      │
  │                                                        └────────┬─────────┘
  │                                                                 │
  │                                                                 ▼
  │                                                        ┌──────────────────┐
  │                                                        │ Curator Node      │
  │                                                        │ score ≥ 0.4       │
  │                                                        │ max 30 per cat.   │
  │                                                        │ → curated_*_data  │
  │                                                        │ → references      │
  │                                                        └────────┬─────────┘
  │                                                                 │
  │                                                                 ▼
  │                                                        ┌──────────────────┐
  │                                                        │ Enricher Node     │
  │                                                        │ Tavily extract()  │
  │                                                        │ adds raw_content  │
  │                                                        │ to curated docs   │
  │                                                        └────────┬─────────┘
  │                                                                 │
  │                                                                 ▼
  │                                                        ┌──────────────────┐
  │                                                        │ Briefing Node     │
  │                                                        │ Gemini 2.5 Flash  │
  │                                                        │ → *_briefing str  │
  │                                                        └────────┬─────────┘
  │                                                                 │
  │                                                                 ▼
  │                                                        ┌──────────────────┐
  │                                                        │ Editor Node       │
  │                                                        │ GPT-4o            │
  │                                                        │ 1. compile report │
  │                                                        │ 2. dedup sweep    │
  │                                                        │ 3. stream chunks  │
  │                                                        │ → report (md)     │
  │                                                        └────────┬─────────┘
  │                                                                 │
  │                                                    job_status[job_id].status = "completed"
  │                                                    mongodb.update_job() + store_report()
  │                                                                 │
  │   ◄── SSE: {type:"complete", report:"..."}  ───────────────────┘
  │
  ├─ POST /generate-pdf  {report_content, company_name}
  │   ◄── PDF stream
```

---

## Entity Relationship Overview

```
ResearchRequest  ──creates──►  Job (job_status entry + MongoDB jobs doc)
     │                               │
     │                               ├─ has InputState fields
     │                               │
     │                               └─ produces ResearchState
                                              │
                       ┌──────────────────────┼──────────────────────┐
                       │                      │                      │
               site_scrape             *_data dicts           messages
               (company crawl)   (4 categories, each        (AIMessage log)
                       │          URL → Document)
                       │                      │
                       │              curated_*_data
                       │          (filtered + sorted)
                       │                      │
                       │              enriched in-place
                       │          (raw_content added)
                       │                      │
                       │                 *_briefing
                       │            (Gemini summaries)
                       │                      │
                       └──────────────────────┤
                                              │
                                         report (str)
                                         references [URL]
                                         reference_titles {URL→title}
                                         reference_info {URL→meta}
                                              │
                                   MongoDB reports doc
```

Each `*_data` dict uses the document URL as the key:

```
Dict[url: str] → Document
  ├── url
  ├── title
  ├── content          (search snippet)
  ├── raw_content      (added by Enricher)
  ├── query
  ├── source           ("web_search" | "company_website")
  ├── score            (Tavily 0–1)
  ├── doc_type         ("company" | "financial" | "news" | "industry")
  └── evaluation
        ├── overall_score
        └── query
```
