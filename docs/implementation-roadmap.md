# Implementation Roadmap

> **Enterprise Knowledge Intelligence Platform — Master Engineering Roadmap**
> Ten-Phase Staged Implementation Plan, Acceptance Criteria, Staged Infrastructure, and Deliverables

---

## Table of Contents

1. [Roadmap Overview & Phasing Strategy](#1-roadmap-overview--phasing-strategy)
2. [Phase Dependency Graph](#2-phase-dependency-graph)
3. [Phase 0: Architecture and Planning (CURRENT)](#3-phase-0-architecture-and-planning-current)
4. [Phase 1: Project Foundation & Relational Core](#4-phase-1-project-foundation--relational-core)
5. [Phase 2: Authentication, Multi-Tenancy & RBAC](#5-phase-2-authentication-multi-tenancy--rbac)
6. [Phase 3: Document Ingestion Pipeline & Synthetic Dataset](#6-phase-3-document-ingestion-pipeline--synthetic-dataset)
7. [Phase 4: Dense Vector Retrieval Service & Vector RAG](#7-phase-4-dense-vector-retrieval-service--vector-rag)
8. [Phase 5: Knowledge Graph Construction & Traversal Service](#8-phase-5-knowledge-graph-construction--traversal-service)
9. [Phase 6: Hybrid Retrieval, Multi-Hop Reasoning & Evidence Fusion](#9-phase-6-hybrid-retrieval-multi-hop-reasoning--evidence-fusion)
10. [Phase 7: LangGraph Workflow Orchestration](#10-phase-7-langgraph-workflow-orchestration)
11. [Phase 8: Complete Enterprise Frontend Integration](#11-phase-8-complete-enterprise-frontend-integration)
12. [Phase 9: Comprehensive Testing, Security Suite & Benchmark Evaluation](#12-phase-9-comprehensive-testing-security-suite--benchmark-evaluation)
13. [Phase 10: Production Deployment, CI/CD & Placement Interview Handbook](#13-phase-10-production-deployment-cicd--placement-interview-handbook)
14. [Staged Infrastructure & Hardware Memory Budget](#14-staged-infrastructure--hardware-memory-budget)
15. [Master Environment Configuration Reference](#15-master-environment-configuration-reference)

---

## 1. Roadmap Overview & Phasing Strategy

The Enterprise Knowledge Intelligence Platform is engineered as a production-grade enterprise system. To maintain stability, architectural integrity, and verifiable milestones on a single 16 GB development laptop (Intel Core Ultra 7 255H), the implementation follows four key principles:

1. **Incremental Verification:** No phase begins until the preceding phase meets its Definition of Done (DoD) and passes its acceptance tests.
2. **Frontend Sequencing:** Rather than leaving all frontend work to the end, UI components are developed incrementally alongside backend capabilities:
   - *Phase 1:* Project foundation, layout, and health status.
   - *Phase 2:* Authentication UI, registration, login, and protected routes.
   - *Phase 3:* Document upload, listing, filtering, and ACL management drawer.
   - *Phase 4:* AI chat interface with citation pills and markdown.
   - *Phase 5:* Knowledge Graph Explorer foundation with React Flow.
   - *Phase 8:* Complete frontend integration, administration, audit log viewer, and polish.
3. **Standalone Retrieval Services vs. Orchestration:**
   - Standalone retrieval modules (`vector_search.py`, `graph_search.py`, `reranker.py`) are implemented and tested independently in Phases 4, 5, and 6.
   - Phase 7 (LangGraph) orchestrates these pre-verified services rather than embedding raw retrieval logic inside graph nodes.
4. **Staged Infrastructure:** Heavy Docker containers are introduced in stages:
   - *Phase 1:* PostgreSQL only (~512 MB – 1 GB)
   - *Phase 4:* Add Qdrant (~512 MB – 1 GB)
   - *Phase 5:* Add Neo4j (~1.0 GB – 1.5 GB)
   - *Phase 10 / Production:* Full multi-container stack.

---

## 2. Phase Dependency Graph

```mermaid
flowchart TD
    P0["Phase 0: Architecture & Planning"] --> P1["Phase 1: Project Foundation (PostgreSQL + React Base)"]
    P1 --> P2["Phase 2: Authentication & Multi-Tenancy (RLS + Auth UI)"]
    P2 --> P3["Phase 3: Document Ingestion Pipeline (Parsers, Storage + Doc UI)"]
    P3 --> P4["Phase 4: Vector Retrieval Service (Qdrant + Chat UI)"]
    P3 --> P5["Phase 5: Knowledge Graph Service (Neo4j + Graph UI)"]
    P4 & P5 --> P6["Phase 6: Hybrid Retrieval & Fusion Service (RRF & Pre-Prompt Check)"]
    P6 --> P7["Phase 7: LangGraph Orchestration (State Machine & Guardrails)"]
    P7 --> P8["Phase 8: Complete Frontend Integration & Admin Views"]
    P8 --> P9["Phase 9: Comprehensive Testing & Security Suite (12 Scenarios)"]
    P9 --> P10["Phase 10: Deployment, CI/CD & Interview Handbook PDF"]

    style P0 fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style P1 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style P2 fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style P3 fill:#ede7f6,stroke:#512da8,stroke-width:2px
    style P4 fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    style P5 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style P6 fill:#e0f2f1,stroke:#00796b,stroke-width:2px
    style P7 fill:#efebe9,stroke:#5d4037,stroke-width:2px
    style P8 fill:#e8eaf6,stroke:#303f9f,stroke-width:2px
    style P9 fill:#ffebee,stroke:#d32f2f,stroke-width:2px
    style P10 fill:#fffde7,stroke:#fbc02d,stroke-width:2px
```

---

## 3. Phase 0: Architecture and Planning (CURRENT)

- **Objective:** Finalize all technical architecture, schema specifications, security controls, API contracts, and implementation roadmaps prior to code execution.
- **Deliverables:**
  - `docs/scratch/core-decisions.md` (Authoritative single source of truth)
  - `docs/architecture.md` (System architecture, Mermaid diagrams, ADRs)
  - `docs/database-schema.md` (PostgreSQL 17 tables, Qdrant, Neo4j, composite keys)
  - `docs/security-design.md` (Multi-tenant isolation, RLS, Argon2id, prompt injection)
  - `docs/api-documentation.md` (REST API contracts, Pydantic models)
  - `docs/implementation-roadmap.md` (This document)
  - `docs/DEVELOPMENT_LOG.md` (Accurate audit log of Phase 0 work)
  - `docs/IMPLEMENTATION_STATUS.md` (Plain-text tracking matrix)
- **Acceptance Criteria:**
  - [x] Workspace root confirmed at `C:\Projects\enterprise-knowledge-intelligence`.
  - [x] Git repository verified on `main` branch with clean working tree.
  - [x] Toolchain verified: Python 3.12, Node.js v26.13, npm 11.20, Docker Desktop on host.
  - [x] All 8 Phase 0 documents reconciled and cross-checked for consistency.
  - [ ] Formal review and explicit sign-off from user received.
- **Definition of Done:** Zero application code implemented, zero dependencies installed, documentation complete and approved.

---

## 4. Phase 1: Project Foundation & Relational Core

- **Objective:** Establish the repository structure, configure Python 3.12 and React + Vite toolchains, initialize PostgreSQL 16 via Docker, create SQLAlchemy models for all 17 tables, and verify Alembic migrations.
- **Key Tasks:**
  1. Create directory structure: `frontend/`, `backend/`, `infrastructure/`, `data/documents/`, `tests/`.
  2. Configure `infrastructure/docker-compose.yml` defining the `postgres` service with health checks and persistent volume `pgdata`.
  3. Initialize backend virtual environment and pin `requirements.txt` (FastAPI, SQLAlchemy, Alembic, Pydantic, uvicorn, asyncpg).
  4. Implement `backend/app/core/config.py` using `pydantic-settings` to load `.env`.
  5. Create declarative SQLAlchemy models for all 17 tables matching `docs/database-schema.md` with composite foreign keys.
  6. Initialize Alembic, generate initial migration `001_initial_schema.py`, and apply to PostgreSQL.
  7. Implement FastAPI application factory with CORS middleware and health endpoints (`/api/v1/health/live`, `/ready`).
  8. Initialize React + Vite + TypeScript frontend with Tailwind CSS, base layout shell, and basic Axios client.
  9. Add comprehensive `.gitignore` ensuring zero secrets, virtualenvs, or node_modules are committed.
- **Frontend Scope in Phase 1:** App shell, navigation header, and backend health status indicator.
- **Git Commit Boundaries:**
  - `chore: initialize repository layout and gitignore`
  - `feat: add docker-compose for postgresql`
  - `feat: add sqlalchemy models for 17 relational tables`
  - `feat: add alembic initial migration and rls setup`
  - `feat: initialize fastapi backend with health endpoints`
  - `feat: initialize react vite frontend with tailwind`
- **Definition of Done:** `npm run dev` serves frontend, `uvicorn` serves backend, PostgreSQL container runs, Alembic runs clean, and `GET /api/v1/health/ready` returns `200 OK`.

---

## 5. Phase 2: Authentication, Multi-Tenancy & RBAC

- **Objective:** Implement complete enterprise authentication, stateful refresh token families, Argon2id hashing, organization tenant isolation with transaction-scoped PostgreSQL RLS, and RBAC dependencies.
- **Key Tasks:**
  1. Implement password security utility using `argon2-cffi` (`time_cost=3`, `memory_cost=65536`) with concurrency semaphore.
  2. Implement JWT encoder/decoder (15-min access tokens) and refresh token manager with `family_id` rotation and 10-second grace window in `refresh_tokens`.
  3. Build authentication routes: `/api/v1/auth/register`, `/login`, `/refresh`, `/logout`, `/users/me`.
  4. Seed 5 system roles (`org_admin`, `doc_manager`, `analyst`, `employee`, `viewer`) and permission mappings.
  5. Seed 3 demo organizations: Finova Technologies, Nova Healthcare, Apex Financial.
  6. Build FastAPI dependencies: `get_current_user`, `get_current_membership`, and `require_permission`.
  7. Enforce transaction-scoped RLS via `SET LOCAL app.current_tenant_id = :tenant_id` on all tenant queries.
  8. Implement frontend login and registration pages with protected route wrappers in React Router.
- **Frontend Scope in Phase 2:** Login page, registration page, auth context provider, JWT interceptor, protected routes.
- **Acceptance Criteria:**
  - [ ] User registration creates user; org bootstrap makes user `org_admin`.
  - [ ] Login returns 15-min access token and rotated refresh cookie.
  - [ ] Consumed refresh token reuse revokes the entire token family.
  - [ ] PostgreSQL queries across tenants return zero rows via forced RLS.
  - [ ] Unit tests for authentication, hashing, and token family rotation pass.
- **Definition of Done:** Automated test verifies Tenant A user cannot access Tenant B records under any query.

---

## 6. Phase 3: Document Ingestion Pipeline & Synthetic Dataset

- **Objective:** Build the secure file upload gauntlet, format parsers (PDF, DOCX, PPTX, TXT), text cleaning, structure-aware chunking, durable deletion cascade, and generate the 20–30 document synthetic dataset.
- **Key Tasks:**
  1. Implement file upload validation (magic bytes, MIME type, 50 MB limit, SHA-256 deduplication).
  2. Implement local managed filesystem storage (`data/documents/{org_id}/{doc_id}/v{version}/`).
  3. Implement format parsers:
     - PDF: PyMuPDF (`fitz`) extracting page text and table bounding boxes.
     - DOCX: `python-docx` extracting paragraphs and tabular structures.
     - PPTX: `python-pptx` extracting slides and speaker notes.
     - TXT: Native UTF-8 normalization.
  4. Implement structure-aware semantic chunker (500–800 tokens, 50–100 token overlap).
  5. Implement document CRUD endpoints, version tracking, authenticated download (`/download`), and fine-grained `document_acl` assignment.
  6. Implement durable deletion cascade worker claiming tasks from `document_cleanup_tasks`.
  7. Generate synthetic enterprise dataset: 20–30 realistic, non-sensitive documents across Finova Technologies, Nova Healthcare, and Apex Financial with interconnected entities.
  8. Build frontend document management interface (drag-and-drop upload, document listing, ACL drawer, delete confirmation).
- **Frontend Scope in Phase 3:** Document upload modal, document table with status badges, ACL management drawer.
- **Acceptance Criteria:**
  - [ ] All 4 file formats parse with page numbers and section metadata preserved.
  - [ ] Uploaded documents create `pending` records and track stage progress.
  - [ ] Deleting a document immediately denies access via soft-delete in PostgreSQL.
  - [ ] Synthetic dataset files generated and committed in `datasets/demo/`.
- **Definition of Done:** Synthetic dataset documents parse and chunk cleanly with zero data loss.

---

## 7. Phase 4: Dense Vector Retrieval Service & Vector RAG

- **Objective:** Configure Qdrant in Docker, implement local CPU embedding generation (`BAAI/bge-small-en-v1.5`), index chunks with tenant/ACL payloads, build standalone vector search service, and build the initial grounded chat interface.
- **Key Tasks:**
  1. Add `qdrant` container to `docker-compose.yml` (ports 6333/6334, memory limit 1 GB).
  2. Create Qdrant collection `enterprise_document_chunks` (Cosine, 384 dim).
  3. Configure Qdrant payload schema indexes (`tenant_id`, `is_public_within_tenant`, `allowed_roles`, `allowed_users`, `department`).
  4. Implement embedding service using `sentence-transformers` locally on CPU.
  5. Implement standalone `backend/app/retrieval/vector_search.py` with mandatory server-side tenant and ACL filtering using `min_should` semantics.
  6. Implement LLM provider abstraction and integrate Google Gemini API.
  7. Construct grounded prompt with cryptographic nonce delimiters framing untrusted chunks.
  8. Implement `/api/v1/chat` and conversation management endpoints for vector-only retrieval.
  9. Build React conversational UI with markdown rendering and clickable citation pills.
- **Frontend Scope in Phase 4:** Chat interface, conversation history sidebar, message bubbles, citation drawer.
- **Acceptance Criteria:**
  - [ ] Standalone vector search service tests cleanly via Pytest.
  - [ ] Unauthorized chunks are never returned by Qdrant search.
  - [ ] LLM generates answers referencing specific document citations (page and snippet).
  - [ ] Baseline retrieval metrics (Recall@K, Precision@K) measured on demo dataset.
- **Definition of Done:** User can query an uploaded document and receive a grounded answer with valid citations.

---

## 8. Phase 5: Knowledge Graph Construction & Traversal Service

- **Objective:** Configure Neo4j in Docker, build entity and relationship extraction using spaCy and structured Gemini outputs, implement tenant-scoped entity resolution, populate property graph with multi-evidence provenance, build standalone graph search service, and build the React Flow graph explorer.
- **Key Tasks:**
  1. Add `neo4j` container to `docker-compose.yml` (ports 7474/7687, heap max 1 GB, pagecache 256 MB).
  2. Establish Neo4j async driver connection pool and apply uniqueness constraints on node IDs.
  3. Implement spaCy pipeline (`en_core_web_sm`) for initial linguistic named entity recognition.
  4. Implement structured Gemini prompt returning validated Pydantic models for business entities and relationships.
  5. Implement entity resolution: normalize case, resolve aliases, deduplicate entities strictly within tenant scope.
  6. Populate Neo4j graph nodes and relationships using explicit `:Evidence` nodes linking chunks to relationships/entities, supporting multi-evidence provenance and fine-grained lifecycle management.
  7. Implement standalone `backend/app/retrieval/graph_search.py` executing parameterized Cypher with tenant constraints.
  8. Build graph query endpoints: `/api/v1/graph/entities`, `/relationships`, `/subgraph`.
  9. Build React Flow frontend Knowledge Graph Explorer with interactive node selection and entity inspector.
- **Frontend Scope in Phase 5:** React Flow graph explorer, entity search filter, node inspector pane.
- **Acceptance Criteria:**
  - [ ] Ingested documents populate 12 node types (9 domain entities including `:Concept`, and 3 structural/provenance labels: `:Document`, `:Chunk`, `:Evidence`) and 11 relationship types.
  - [ ] Relationships retain multi-evidence provenance via explicit `:Evidence` nodes; deleting one document deletes only its evidence nodes and prunes orphaned edges while preserving shared domain entities.
  - [ ] Standalone graph traversal service passes Pytest test suite.
  - [ ] Subgraph endpoint renders cleanly in React Flow with distinct entity color badges.
- **Definition of Done:** Knowledge graph reflects multi-document entity relationships present in the synthetic dataset.

---

## 9. Phase 6: Hybrid Retrieval, Multi-Hop Reasoning & Evidence Fusion

- **Objective:** Implement standalone Reciprocal Rank Fusion (RRF), query intent classification, multi-hop reasoning, and authoritative pre-prompt PostgreSQL permission revalidation.
- **Key Tasks:**
  1. Implement standalone `backend/app/retrieval/reranker.py` implementing Reciprocal Rank Fusion (RRF).
  2. Implement query intent classifier: routes queries to `VECTOR`, `GRAPH`, or `HYBRID`.
  3. Implement multi-hop Cypher traversal supporting up to 3 hops with mandatory `{tenant_id: $tenant_id}` on all nodes.
  4. Implement graph path chunk retriever: fetches original supporting text chunks for graph edges.
  5. Implement authoritative pre-prompt permission revalidation: cross-checks candidate evidence against PostgreSQL live status.
  6. Implement multi-hop answer generation prompt capable of synthesizing cross-document paths.
  7. Execute **Experiment 1:** Benchmark Vector-only vs. Graph-only vs. Hybrid GraphRAG on multi-hop questions.
- **Acceptance Criteria:**
  - [ ] System answers complex multi-hop questions (e.g. Project Alpha risk -> mitigation strategy -> approving CFO).
  - [ ] Standalone hybrid retrieval and fusion tests cleanly via Pytest.
  - [ ] Soft-deleted documents or revoked ACLs are purged in `validate_access` before reaching the LLM prompt.
  - [ ] Experiment 1 results recorded with factual measurements.
- **Definition of Done:** Demonstration question 7 produces a grounded, multi-hop answer citing multiple source documents.

---

## 10. Phase 7: LangGraph Workflow Orchestration

- **Objective:** Encapsulate the entire query lifecycle into a formal LangGraph state machine with 12 modular nodes, typed state (`RAGState`), conditional edges, bounded retries, and safe fallbacks, calling the pre-verified retrieval services.
- **Key Tasks:**
  1. Define `RAGState` TypedDict containing query, user session, retrieval candidates, authorized evidence, and citations.
  2. Implement 12 modular workflow nodes delegating to the retrieval services built in Phases 4–6.
  3. Configure conditional routing branching on query classification.
  4. Implement retry loops with exponential backoff for external LLM API calls.
  5. Implement graceful degradation: if Neo4j is unavailable, route safely to vector-only with user notification (`fallback_applied: true`).
  6. Implement citation validation node: verifies claims against chunk coordinates.
  7. Wire LangGraph workflow directly into `POST /api/v1/chat`.
- **Acceptance Criteria:**
  - [ ] Workflow executes end-to-end within a single graph invocation.
  - [ ] Conditional routing selects correct retrieval strategy based on query complexity.
  - [ ] Safe fallback works during simulated Neo4j container outage.
  - [ ] Zero unauthorized data enters the context assembly node.
- **Definition of Done:** Complete automated integration test runs chat queries through the LangGraph workflow.

---

## 11. Phase 8: Complete Enterprise Frontend Integration

- **Objective:** Finalize a polished, responsive enterprise SaaS user interface integrating real backend endpoints across all modules with complete permission awareness.
- **Key Tasks:**
  1. Implement Organization Analytics Dashboard with Recharts (document counts, status breakdown, query volume).
  2. Complete Document Management module: drag-and-drop upload, filtering, version drawer, ACL management modal.
  3. Finalize Conversational AI Assistant: markdown rendering, copy controls, citation drawer, feedback rating controls.
  4. Finalize Knowledge Graph Explorer: React Flow viewport, entity search, node filters, subgraph expansion.
  5. Implement Administration module: user management, role assignment, organization invitations, audit log viewer.
  6. Implement historical conversation citation revalidation UI: displays badge if citation has been revoked.
  7. Apply cohesive enterprise design system with loading skeletons, empty states, and toast notifications.
- **Acceptance Criteria:**
  - [ ] Zero hardcoded mockup data in production views; all components consume `/api/v1` endpoints.
  - [ ] UI elements (e.g. Admin tab, Delete button) conditionally render based on authenticated user roles.
  - [ ] Error toasts display user-friendly messages for 400, 403, 404, 429 errors.
- **Definition of Done:** Frontend runs cleanly on host port 5173 with seamless end-to-end API communication.

---

## 12. Phase 9: Comprehensive Testing, Security Suite & Benchmark Evaluation

- **Objective:** Execute full unit and integration test suites, run the 12-scenario security test suite, benchmark retrieval and answer quality, execute Experiments 2 and 3, and compile the formal evaluation report.
- **Key Tasks:**
  1. Implement Pytest suite covering FastAPI endpoints, parsers, chunker, and security dependencies.
  2. Implement Vitest frontend suite covering auth context, API client, and key UI components.
  3. Execute **12-Scenario Security Test Suite** (cross-tenant access, role escalation, UUID probing, Cypher injection, prompt injection, immediate ACL revocation, stale index scrubbing, conversation snooping, unauthenticated download, ingestion snooping, RLS connection-pool safety, cleanup worker failure).
  4. Benchmark retrieval metrics: Recall@K, Precision@K, Mean Reciprocal Rank (MRR), nDCG.
  5. Benchmark answer quality: Faithfulness, Answer Relevance, Citation Precision/Recall.
  6. Execute **Experiment 2:** Tenant isolation and ACL revocation verification.
  7. Execute **Experiment 3:** Chunk size (500 vs. 800 tokens) and top-K parameter sensitivity.
  8. Measure actual CPU, memory, and P50/P95 latency benchmarks.
  9. Compile formal evaluation results into `docs/evaluation-report.md`.
- **Acceptance Criteria:**
  - [ ] 12 out of 12 security test cases pass with ZERO unauthorized data disclosures.
  - [ ] Unit and integration test coverage meets production standards.
  - [ ] All benchmark metrics recorded from actual test executions without fabrication.
- **Definition of Done:** Security test suite passes completely with zero failures.

---

## 13. Phase 10: Production Deployment, CI/CD & Placement Interview Handbook

- **Objective:** Create production Docker Compose configurations, GitHub Actions CI/CD workflows, project README, comprehensive Markdown sources, and generate the final Placement Interview Handbook PDF.
- **Key Tasks:**
  1. Create production `infrastructure/docker-compose.prod.yml` with health checks and restart policies.
  2. Create GitHub Actions CI workflow (`.github/workflows/ci.yml`) for linting, typing (`mypy`, `tsc`), and test runs.
  3. Write comprehensive root `README.md` with architectural overview, quickstart commands, and API documentation.
  4. Write editable Markdown sources for the 18 chapters of the Placement Interview Handbook under `docs/interview-handbook/`.
  5. Implement ReportLab PDF generation script (`scripts/generate_handbook_pdf.py`).
  6. Generate the final PDF: `Enterprise_Knowledge_Intelligence_Platform_Interview_Handbook.pdf`.
  7. Verify PDF: 18 chapters, clickable TOC, formatted code blocks, architecture diagrams, zero clipped text.
- **Acceptance Criteria:**
  - [ ] Application starts cleanly from scratch using documented commands.
  - [ ] GitHub Actions CI pipeline passes.
  - [ ] Interview Handbook PDF generated, verified, and downloadable.
  - [ ] Final project sign-off achieved.
- **Definition of Done:** Project is completely functional, documented, and ready for technical portfolio demonstrations.

---

## 14. Staged Infrastructure & Hardware Memory Budget

```mermaid
flowchart LR
    subgraph STAGE_1["Phase 1: Relational Core"]
        PG["PostgreSQL 16 (512 MB – 1 GB)"]
    end

    subgraph STAGE_2["Phase 4: Vector RAG"]
        PG2["PostgreSQL 16 (512 MB – 1 GB)"]
        QD["Qdrant (512 MB – 1 GB)"]
        PG2 --- QD
    end

    subgraph STAGE_3["Phase 5+: Hybrid GraphRAG"]
        PG3["PostgreSQL 16 (1 GB)"]
        QD2["Qdrant (1 GB)"]
        N4J["Neo4j 5.x (1.5 GB)"]
        PG3 --- QD2 --- N4J
    end

    STAGE_1 --> STAGE_2 --> STAGE_3
```

- **Target Machine:** Windows 11, WSL2 / Docker Desktop, Intel Core Ultra 7 255H, 16 GB RAM.
- **Estimated Peak Container Memory:** ~3.5 GB RAM total across PostgreSQL, Qdrant, and Neo4j.
- **Estimated Host Usage:** ~1.5 GB RAM for FastAPI + embedding process; ~300 MB for Vite dev server.
- **Safety Headroom:** ~3 GB free RAM, preventing memory pressure. Actual usage will be measured and documented during benchmarking.

---

## 15. Master Environment Configuration Reference

The following environment variables are required across the platform lifecycle. Documented in `.env.example`:

```bash
# === Application Environment ===
APP_ENV=development
LOG_LEVEL=INFO
SECRET_KEY=generate-a-secure-random-64-character-hex-key
CORS_ORIGINS=["http://localhost:5173"]

# === PostgreSQL Relational Database ===
DATABASE_URL=postgresql+asyncpg://ekip_app:CHANGE_ME@localhost:5432/ekip_dbPOSTGRES_USER=ekip_app
POSTGRES_PASSWORD=CHANGE_ME
POSTGRES_DB=ekip_db
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# === Qdrant Vector Store ===
QDRANT_URL=http://localhost:6333
QDRANT_COLLECTION=enterprise_document_chunks
QDRANT_API_KEY=

# === Neo4j Knowledge Graph ===
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=CHANGE_ME

# === JWT Authentication ===
JWT_SECRET_KEY=generate-a-secure-random-64-character-hex-key
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# === Embedding Configuration ===
EMBEDDING_PROVIDER=local
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
EMBEDDING_DIMENSION=384

# === LLM Provider Configuration ===
LLM_PROVIDER=gemini
LLM_MODEL=gemini-2.0-flash
LLM_API_KEY=your-gemini-api-key-here
LLM_TEMPERATURE=0.0
LLM_MAX_TOKENS=2048

# === File Storage & Ingestion Limits ===
DOCUMENT_STORAGE_PATH=./data/documents
MAX_UPLOAD_SIZE_MB=50
```
