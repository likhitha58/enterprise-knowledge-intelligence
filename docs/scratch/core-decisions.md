# Core Architectural Decisions Reference
# Authoritative single source of truth for all Phase 0 specifications — Enterprise Knowledge Intelligence Platform

---

## 1. Multi-Tenant Model, Tenant Integrity & Authorization Authority

- **Tenant Identity:** Organization-based multi-tenancy. Every tenant is represented by a unique `organization_id` in PostgreSQL.
- **Tenant Context Resolution:** 
  1. The client presents a JWT access token in the `Authorization` header.
  2. For multi-organization users, an `X-Organization-ID` header provides a *membership selection hint*.
  3. FastAPI validates that the authenticated user holds an active membership (`is_active = true`) in the specified organization.
  4. Tenant context is derived strictly from server-side database records. Tenant IDs, user IDs, and roles supplied in request bodies or query parameters are NEVER trusted for authorization.
- **Authoritative Boundary:** PostgreSQL is the **sole authoritative system of record** for authentication, organizational membership, role permissions, document metadata, document status, and document ACLs.
  - Qdrant vector payloads and Neo4j graph subgraphs are secondary retrieval indices, NOT permission authorities.
  - All candidates retrieved from Qdrant or Neo4j must be revalidated against live PostgreSQL records before entering LLM context.
- **Relational Tenant Integrity Constraints:**
  - Foreign key constraints enforce relational tenant consistency across all tables:
    - `memberships`: `UNIQUE (organization_id, user_id)` allows composite FKs.
    - `documents`: `UNIQUE (id, organization_id)` allows tenant-scoped child FKs.
    - Document uploader: `FOREIGN KEY (organization_id, uploaded_by) REFERENCES memberships(organization_id, user_id)`.
    - Document ACL target: when `grant_type = 'user'`, composite FK `(organization_id, user_id) REFERENCES memberships(organization_id, user_id)` guarantees the granted user belongs to the document's organization.
    - Ingestion jobs: `FOREIGN KEY (document_id, organization_id) REFERENCES documents(id, organization_id)`.
    - Document versions: `FOREIGN KEY (document_id, organization_id) REFERENCES documents(id, organization_id)`.
    - `documents.current_version_id`: `FOREIGN KEY (id, current_version_id) REFERENCES document_versions(document_id, id)` ensures a document's active version belongs to that document.
    - Conversations: `FOREIGN KEY (organization_id, user_id) REFERENCES memberships(organization_id, user_id)`.
    - Feedback: `FOREIGN KEY (message_id) REFERENCES messages(id)` and user membership validated against the conversation's organization.

---

## 2. PostgreSQL Row-Level Security (RLS) & Table Security Treatment

- **Connection Pool Tenant Safety:**
  - In connection-pooled asynchronous environments (e.g., SQLAlchemy with `asyncpg`), session-level settings can leak across requests on reused physical connections.
  - Therefore, tenant context MUST be set inside an explicit transaction block using transaction-scoped configuration:
    ```sql
    SET LOCAL app.current_tenant_id = :tenant_id;
    -- or SELECT set_config('app.current_tenant_id', :tenant_id::text, true);
    ```
  - When the transaction commits or aborts, `app.current_tenant_id` is automatically cleared.
- **Dedicated Application Role:**
  - The FastAPI application connects using a dedicated role (`ekip_app`) that:
    - Is NOT a `SUPERUSER`.
    - Does NOT have the `BYPASSRLS` attribute.
    - Has only `SELECT`, `INSERT`, `UPDATE`, `DELETE` grants on application tables (and `SELECT`, `INSERT` only on `audit_logs`).
- **RLS Coverage & Security Treatment Across All 17 Tables:**
  1. **Nine Tenant-Owned Tables (RLS Enabled and FORCED):**
     - `documents`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `document_versions`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `document_acl`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `conversations`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `messages`: Scoped via `organization_id` foreign key with forced RLS
     - `feedback`: Scoped via `organization_id` foreign key with forced RLS
     - `ingestion_jobs`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `document_cleanup_tasks`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
     - `audit_logs`: `USING (organization_id = NULLIF(current_setting('app.current_tenant_id', true), '')::uuid)`
  2. **Tenant Management Tables (Explicit Query Scoping):**
     - `organizations`: Access is restricted strictly to an authenticated user's memberships:
       `SELECT o.* FROM organizations o JOIN memberships m ON o.id = m.organization_id WHERE m.user_id = :auth_user_id AND m.is_active = true AND o.is_active = true`. A user cannot query or view organizations they do not belong to.
     - `memberships`: Access is dual-scoped:
       - Regular users can only query their own memberships: `WHERE user_id = :auth_user_id AND is_active = true`.
       - Organization administrators (`org_admin`) can only query memberships within their active tenant: `WHERE organization_id = :active_tenant_id`. Cross-tenant membership queries are rejected.
     - `user_roles`: Scoped strictly through `memberships` where `membership.organization_id = :active_tenant_id`. Admins can inspect and assign roles only for memberships belonging to their active tenant.
  3. **Global Reference Tables (No RLS, Static System Data):**
     - `users`: Global authentication records. Lookups restricted by `id = :auth_user_id` or unique `email`.
     - `roles`: Read-only system role definitions (`org_admin`, `doc_manager`, `analyst`, `employee`, `viewer`).
     - `permissions`: Read-only atomic functional permissions (`resource:action`).
     - `role_permissions`: Read-only mappings linking system roles to permissions.
  4. **User Security Table:**
     - `refresh_tokens`: Scoped strictly to the authenticated user ID and token hash:
       `WHERE user_id = :auth_user_id AND token_hash = :hash`. Refresh tokens never carry or cross tenant boundaries.

---

## 3. Authentication & Refresh Token Concurrency Management

- **Password Hashing:**
  - Argon2id via `argon2-cffi`.
  - Parameters: `time_cost=3`, `memory_cost=65536` KiB (64 MB), `parallelism=4`, salt length=16 bytes, hash length=32 bytes.
  - Concurrency control: Hashing operations are bounded by a server-side semaphore (maximum 4 concurrent hashes) to prevent memory exhaustion under load.
- **Access Tokens:**
  - Short-lived JWTs (15-minute TTL).
  - Claims: `sub` (user_id), `exp`, `iat`, `jti`.
  - Contains NO `tenant_id` and NO role claims. Tenant context and effective permissions are loaded dynamically per request from PostgreSQL.
- **Refresh Token Lifecycle & Concurrency Handling:**
  - Cryptographically random 256-bit opaque tokens (7-day TTL).
  - Stored strictly as SHA-256 hashes in `refresh_tokens`.
  - Tracking fields: `id`, `user_id`, `token_hash`, `family_id`, `is_revoked`, `revoked_at`, `replaced_by_token_hash`, `expires_at`, `created_at`, `last_used_at`.
  - **Client-Side Single-Flight Mutex (Primary Protection):**
    - The frontend Axios interceptor implements a single-flight mutex/promise latch.
    - When an access token expires, the first `401 Unauthorized` triggers the refresh request. All subsequent concurrent API requests are paused and subscribe to the single in-flight refresh promise, awaiting the new access token.
    - This eliminates multiple simultaneous refresh requests from the same client session.
  - **Backend Serialized Refresh Operations:**
    - The backend executes refresh token rotation inside an explicit transaction with row-level locking:
      ```sql
      SELECT * FROM refresh_tokens
      WHERE token_hash = :hash
      FOR UPDATE;
      ```
    - The active token is marked `is_revoked = true`, `revoked_at = now()`, and `replaced_by_token_hash = :new_hash`.
    - A newly generated refresh token is inserted into `refresh_tokens` sharing the same `family_id`.
  - **Strict Replay Detection & Immediate Family Revocation:**
    - Refresh tokens are strictly single-use. No arbitrary reuse of consumed tokens is permitted.
    - If a request presents a token where `is_revoked = true`, it is classified as a token theft or replay attack.
    - The backend immediately executes **token family revocation**:
      ```sql
      UPDATE refresh_tokens
      SET is_revoked = true, revoked_at = now()
      WHERE family_id = :family_id;
      ```
    - All active sessions for that token family are terminated immediately, and an alert is recorded in `audit_logs`.
- **Delivery & CSRF Protection:**
  - Refresh tokens delivered to browsers via `HttpOnly`, `Secure`, `SameSite=Lax` cookie (`ekip_refresh_token`).
  - Mutating requests using cookie authentication require `Origin` / `Referer` validation and a custom CSRF protection header (`X-CSRF-Token`).
  - Standard JSON response delivery supported for non-browser/mobile clients.

---

## 4. Authorization: RBAC and Document-Level ACL

- **5 System Roles:**
  1. `org_admin`: Full tenant administrative control (users, memberships, audit logs, all documents).
  2. `doc_manager`: Document lifecycle management (upload, versioning, ACL updates, deletion).
  3. `analyst`: Can upload documents, run hybrid/graph queries, view knowledge graph.
  4. `employee`: Can upload own documents, query knowledge base within permitted ACLs.
  5. `viewer`: Read-only queries over explicitly permitted documents; cannot upload.
- **Permission Inheritance Hierarchy:**
  - `read`: Permits metadata viewing, content reading, and inclusion in vector/graph RAG retrieval.
  - `write`: Inherits `read`; permits document metadata updates and uploading new versions.
  - `manage`: Inherits `read` and `write`; permits editing document ACL grants.
- **Document ACL Semantics:**
  - Evaluated on top of RBAC. RBAC determines functional capability; ACL determines document visibility.
  - Target XOR constraint: Exactly ONE of `role_id` or `user_id` must be NOT NULL per ACL entry.
  - Organizational constraint: `role_id` and `user_id` must belong to the document's `organization_id`.
- **Classification vs. Authorization:**
  - Classification levels: `public`, `internal`, `confidential`, `restricted`.
  - **Classification is NOT an authorization bypass.** A document does not become retrievable simply because it is marked `public`.
  - Document retrieval requires:
    1. Active tenant membership.
    2. Functional capability (`documents:read` or `chat:query`).
    3. Document read clearance (via `org_admin` privilege, uploader ownership, explicit role ACL, explicit user ACL, or tenant-wide public flag).
- **FastAPI vs. LangGraph Authorization Responsibilities:**
  1. *FastAPI Ingress:* Authenticates JWT, verifies active user status, validates active membership in `X-Organization-ID`, resolves roles, sets transaction RLS context (`SET LOCAL app.current_tenant_id`).
  2. *LangGraph Orchestration:* Receives trusted context (user_id, tenant_id, roles). Executes retrieval via Qdrant/Neo4j with filter hints.
  3. *Authoritative Pre-Prompt Check (`validate_access`):* LangGraph revalidates all retrieved candidates against live PostgreSQL ACLs and status (`is_deleted = false`, `status = 'ready'`). Unauthorized evidence is purged before context construction.

---

## 5. PostgreSQL Relational Schema (17 Tables)

1. `organizations`: Tenant registry.
2. `users`: Global user accounts.
3. `memberships`: User-to-organization enrollment. `UNIQUE (organization_id, user_id)`.
4. `roles`: System role catalogue.
5. `permissions`: Atomic functional permissions.
6. `role_permissions`: Role-permission mappings.
7. `user_roles`: Membership-scoped role assignments.
8. `refresh_tokens`: Stateful token sessions with family tracking and reuse detection.
9. `documents`: Document metadata, status, tenant ownership. `UNIQUE (id, organization_id)`.
10. `document_versions`: Revision history. `UNIQUE (document_id, id)` and `UNIQUE (document_id, version_number)`.
11. `document_acl`: Fine-grained ACL entries with XOR target constraint.
12. `conversations`: Chat session headers.
13. `messages`: Individual chat turns with citations and retrieval metadata.
14. `feedback`: User ratings on assistant messages. `UNIQUE (message_id, user_id)`.
15. `ingestion_jobs`: Multi-stage ingestion progress tracking.
16. `document_cleanup_tasks`: Durable asynchronous cleanup queue. `UNIQUE (document_id, target_system)`.
17. `audit_logs`: Application-managed append-only audit trail with restricted database role privileges (no `UPDATE` or `DELETE` grants).

---

## 6. Qdrant Vector Collection & Mandatory Authorization Filtering

- **Collection Name:** `enterprise_document_chunks`
- **Vector Dimensions:** 384 (matches `BAAI/bge-small-en-v1.5`)
- **Distance Metric:** Cosine
- **Payload Schema:**
  - `tenant_id` (keyword, indexed)
  - `document_id` (keyword, indexed)
  - `document_version_id` (keyword)
  - `chunk_id` (keyword)
  - `chunk_index` (integer, indexed)
  - `page_number` (integer, indexed)
  - `section` (keyword)
  - `is_public_within_tenant` (boolean, indexed)
  - `allowed_roles` (keyword[], indexed)
  - `allowed_users` (keyword[], indexed)
  - `department` (keyword, indexed)
  - `classification` (keyword, indexed — informational metadata)
  - `content_hash` (keyword)
  - `text` (text, stored not indexed)
  - `document_title` (keyword)
  - `created_at` (keyword)
- **Mandatory Authorization Filtering Structure (`min_should` Semantics):**
  - Every vector query MUST enforce both tenant isolation and at least one authorization condition.
  - Authorization conditions are NEVER optional. Using Qdrant's `min_should` semantics, at least ONE access grant must match:
    ```python
    search_filter = models.Filter(
        must=[
            models.FieldCondition(
                key="tenant_id",
                match=models.MatchValue(value=str(session.organization_id))
            ),
            models.Filter(
                should=[
                    models.FieldCondition(
                        key="is_public_within_tenant",
                        match=models.MatchValue(value=True)
                    ),
                    models.FieldCondition(
                        key="allowed_roles",
                        match=models.MatchAny(any=user_roles)
                    ),
                    models.FieldCondition(
                        key="allowed_users",
                        match=models.MatchAny(any=[str(session.user_id)])
                    )
                ],
                min_should=models.MinShould(
                    conditions=[
                        models.FieldCondition(key="is_public_within_tenant", match=models.MatchValue(value=True)),
                        models.FieldCondition(key="allowed_roles", match=models.MatchAny(any=user_roles)),
                        models.FieldCondition(key="allowed_users", match=models.MatchAny(any=[str(session.user_id)]))
                    ],
                    min_count=1
                )
            )
        ]
    )
    ```
  - Note: Classification is NOT used as an authorization filter.
  - **Authoritative Pre-Prompt Check:** This Qdrant filter is an initial retrieval boundary. PostgreSQL remains authoritative for final permission validation in the LangGraph `validate_access` node.

---

## 7. Neo4j Knowledge Graph, Entity Taxonomy & Explicit Evidence Node Model

- **Graph Entity Taxonomy (12 Node Labels):**
  - **Nine Domain Entity Labels:**
    1. `:Organization {id, tenant_id, name}`
    2. `:Person {id, tenant_id, name, title, department}`
    3. `:Department {id, tenant_id, name}`
    4. `:Project {id, tenant_id, name, status, budget}`
    5. `:Risk {id, tenant_id, name, risk_type, severity}`
    6. `:Strategy {id, tenant_id, name, strategy_type}`
    7. `:Event {id, tenant_id, name, date}`
    8. `:Product {id, tenant_id, name, category}`
    9. `:Concept {id, tenant_id, name, description}` (Represents general business terms, technologies, methodologies, and regulatory frameworks, e.g. "Liquidity Buffer", "GDPR").
  - **Three Structural / Provenance Labels:**
    10. `:Document {id, tenant_id, title, classification}`
    11. `:Chunk {id, tenant_id, document_id, chunk_index, page_number}`
    12. `:Evidence {id, tenant_id, source_document_id, source_chunk_id, page_number, section, confidence, extracted_at, relationship_type}`
- **11 Domain Relationship Types:**
  `WORKS_FOR`, `MANAGES`, `BELONGS_TO`, `HAS_RISK`, `ADDRESSED_BY`, `APPROVED_BY`, `MENTIONED_IN`, `CONTAINS`, `RESPONSIBLE_FOR`, `RELATED_TO`, `SUPPORTED_BY`.
- **Explicit Provenance Model via `:Evidence` Nodes:**
  - To guarantee safety under concurrent ingestion and deletion, provenance is modeled as discrete `:Evidence` nodes rather than mutating JSON arrays on edges.
  - Graph Topology:
    - Domain entities connect via direct relationships: `(a:Person)-[r:WORKS_FOR {tenant_id: $tenant_id}]->(b:Organization)`.
    - Each supporting chunk connects to an evidence node: `(c:Chunk)-[:PROVIDES_EVIDENCE]->(e:Evidence {id: $ev_id, tenant_id: $tenant_id, source_document_id: $doc_id, source_chunk_id: $chunk_id, page_number: 4, confidence: 0.95, relationship_type: 'WORKS_FOR'})`.
    - The evidence node links the source and target entities: `(a)-[:HAS_EVIDENCE]->(e)-[:EVIDENCE_FOR]->(b)`.
  - **Deletion Guarantees:**
    - When Document A is deleted:
      ```cypher
      MATCH (e:Evidence {source_document_id: $doc_id, tenant_id: $tenant_id})
      DETACH DELETE e;
      ```
    - The relationship edge between `a` and `b` is deleted ONLY if no remaining `:Evidence` nodes support that connection:
      ```cypher
      MATCH (a)-[r:WORKS_FOR {tenant_id: $tenant_id}]->(b)
      WHERE NOT EXISTS {
          MATCH (a)-[:HAS_EVIDENCE]->(e:Evidence {tenant_id: $tenant_id, relationship_type: 'WORKS_FOR'})-[:EVIDENCE_FOR]->(b)
      }
      DELETE r;
      ```
    - Shared entity nodes (`Person`, `Project`, etc.) are NEVER deleted if other relationships or evidence nodes connect to them.
- **Tenant-Scoped Traversal:**
  - Every Cypher traversal template MUST constrain every node and relationship along the path: `{tenant_id: $tenant_id}`.
  - Freeform or unrestricted LLM-generated Cypher is strictly prohibited.
  - Traversal results return only evidence supported by documents currently accessible to the caller.

---

## 8. Document Lifecycle & Durable Deletion Cascade

- **Staged Ingestion Workflow:**
  1. File validation (magic bytes, MIME, 50 MB limit, SHA-256 duplicate check).
  2. Local storage write (`data/documents/{org_id}/{doc_id}/v1/`).
  3. Relational record creation in `documents` (`status = 'pending'`), `document_versions`, and `document_acl`.
  4. Ingestion job record created in `ingestion_jobs` (`status = 'queued'`).
  5. Parsing (PyMuPDF / python-docx / python-pptx / txt), text cleaning, semantic chunking (500–800 tokens).
  6. Embedding generation on CPU (`bge-small-en-v1.5`, 384 dim).
  7. Vector upsert to Qdrant with mandatory `is_public_within_tenant`, `allowed_roles`, and `allowed_users` tags.
  8. Entity & relationship extraction via spaCy + structured Gemini output; tenant-scoped entity resolution.
  9. Neo4j graph construction with explicit `:Evidence` nodes and domain edges.
  10. Final atomic status update: `documents.status = 'ready'`, `ingestion_jobs.status = 'completed'`.
- **Durable Deletion Cascade:**
  1. `DELETE /api/v1/documents/{id}` immediately executes a soft-delete in PostgreSQL (`is_deleted = true`, `status = 'deleted'`).
  2. Immediate denial: Any query arriving after the soft-delete returns `404 Not Found` via PostgreSQL checks.
  3. Deletion tasks enqueued in `document_cleanup_tasks` (`UNIQUE(document_id, target_system)`):
     - Task for `qdrant`: deletes all points with matching `document_id`.
     - Task for `neo4j`: removes document/chunk/evidence nodes and prunes unsupported edges.
     - Task for `storage`: removes physical on-disk file.
  4. Cleanup worker leases tasks with a lease timeout (`claimed_at`, `claimed_by`) and retries with backoff.

---

## 9. Chat Retrieval Modes & LangGraph Orchestration

- **Retrieval Modes in `POST /api/v1/chat`:**
  - `AUTO` (default): Query classifier inspects complexity. Simple factual queries -> `VECTOR`; entity relationships -> `GRAPH`; multi-hop / cross-document -> `HYBRID`.
  - `VECTOR`: Forces vector-only retrieval via Qdrant.
  - `GRAPH`: Forces graph-only traversal via Neo4j. (Note: Supporting source text chunks are still retrieved and cited).
  - `HYBRID`: Combines vector similarity and graph traversal via Reciprocal Rank Fusion (RRF).
  - If a requested mode is unavailable (e.g. Neo4j is down), the system returns an explicit failure or documented fallback with metadata notification (`fallback_applied: true`). It NEVER silently switches modes without client disclosure.
- **Safety Invariant for `PARTIAL` Answers:**
  - If evidence is excluded due to ACL restrictions, it is silently dropped in `validate_access`.
  - The answer status is set to `PARTIAL`, but the response NEVER mentions the titles, IDs, or snippets of inaccessible documents.

---

## 10. Prompt Injection Defense Architecture

- **Delimiters are NOT a Security Boundary:**
  - Wrapping text in `<evidence_chunk nonce="...">` provides structural framing for the parser, but is NOT an absolute security boundary against advanced jailbreaks.
- **True Multi-Layer Security Boundary:**
  1. *Server-Side Access Control:* Malicious instructions cannot query or access data the user is not cleared to read.
  2. *Constrained Retrieval:* Only authorized chunks enter the context.
  3. *Zero Tool Autonomy:* The LLM has no tools to modify permissions, execute arbitrary database queries, or write data.
  4. *Policy Precedence Directives:* System prompts enforce that document text is passive, untrusted reference material.
  5. *Output & Citation Validation:* All claims and citations are verified against retrieved chunk IDs before delivery.

---

## 11. Staged Infrastructure & Hardware Budget

- **Target Development Machine:** Windows 11, WSL2 / Docker Desktop, Intel Core Ultra 7 255H, 16 GB RAM, integrated graphics (no CUDA).
- **Selective Containerization:**
  - Host process: React Vite dev server, FastAPI backend, local CPU embedding model.
  - Docker containers: PostgreSQL, Qdrant, Neo4j.
- **Staged Startup Sequence:**
  1. *Phase 1:* PostgreSQL 16 only (~512 MB – 1 GB).
  2. *Phase 4:* Add Qdrant (~512 MB – 1 GB).
  3. *Phase 5:* Add Neo4j (~1.0 GB – 1.5 GB).
  4. *Redis:* Optional (Phase 10+); introduced only if background worker scaling requires Celery.
- **Encryption Reality:** Docker named volumes are NOT automatically encrypted. Data-at-rest encryption relies on the host OS / filesystem (BitLocker / LUKS).

---

## 12. Open Decisions for Implementation Review

1. **OCR Engine Selection for Scanned Documents:** Deferred to post-Phase 3; digital documents prioritized initially.
2. **Refresh Token Delivery Default:** Cookie-based (`HttpOnly`, `SameSite=Lax`) as default for browser clients, with JSON body response available for CLI / programmatic API clients.
3. **Graph Subgraph Visualization Density:** Setting max nodes to 100 and depth to 2 as initial frontend rendering defaults in React Flow.
