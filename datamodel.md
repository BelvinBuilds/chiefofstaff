# 0) Extensions & Schema Layout

```sql
-- Enable required extensions (install once per DB)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector
CREATE EXTENSION IF NOT EXISTS btree_gin;   -- optional, for GIN on scalar

-- Logical namespaces
CREATE SCHEMA IF NOT EXISTS core;  -- structured entities (people, deals, etc.)
CREATE SCHEMA IF NOT EXISTS doc;   -- files, chunks, transcripts
CREATE SCHEMA IF NOT EXISTS ai;    -- embeddings, search helpers, jobs
CREATE SCHEMA IF NOT EXISTS ops;   -- approvals, activity logs
CREATE SCHEMA IF NOT EXISTS sec;   -- tenants, users, auth aids (optional)
```

---

# 1) Multi-tenancy & Users (optional but future-proof)

```sql
-- You’re a single user now, but this supports future partners safely.
CREATE TABLE sec.tenant (
  tenant_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sec.app_user (
  user_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  email CITEXT UNIQUE NOT NULL,
  full_name TEXT,
  role TEXT CHECK (role IN ('owner','partner','analyst','ops')) DEFAULT 'owner',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Every table below includes tenant_id to allow Row-Level Security if/when needed.
```

---

# 2) Core Operational Model

```sql
-- Persons (founders, investors, operators)
CREATE TABLE core.person (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  person_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  full_name TEXT NOT NULL,
  email CITEXT,
  role TEXT,                 -- founder, CEO, CTO, etc.
  affiliation TEXT,          -- company or org
  linkedin_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Companies (Michigan-focused startups fit here)
CREATE TABLE core.company (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  company_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  website TEXT,
  sector TEXT,               -- mobility, medtech, climate, etc.
  stage TEXT,                -- pre-seed, seed, A...
  hq_location TEXT,          -- Ann Arbor, Detroit, etc.
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Deals (pipeline)
CREATE TABLE core.deal (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  deal_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_id UUID REFERENCES core.company(company_id) ON DELETE SET NULL,
  status TEXT CHECK (status IN ('new','triage','in_diligence','win','loss','parked')) NOT NULL DEFAULT 'new',
  thesis_tags TEXT[],        -- ['mobility','AI','Michigan']
  source_channel TEXT,       -- outlook, formassembly, intro, event
  round_target TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Interactions (emails, meetings, calls, chats, notes)
CREATE TABLE core.interaction (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  interaction_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  deal_id UUID REFERENCES core.deal(deal_id) ON DELETE SET NULL,
  person_id UUID REFERENCES core.person(person_id) ON DELETE SET NULL,
  type TEXT CHECK (type IN ('email','meeting','chat','call','note')) NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL,
  summary TEXT,
  transcript_doc_id UUID,    -- points to doc.document.file_id if applicable
  action_items JSONB,        -- [{title, owner_person_id, due_date}]
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Tasks (sync with Monday/Outlook)
CREATE TABLE core.task (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  task_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  source TEXT CHECK (source IN ('monday','outlook','ai')) NOT NULL,
  deal_id UUID REFERENCES core.deal(deal_id) ON DELETE SET NULL,
  owner_person_id UUID REFERENCES core.person(person_id) ON DELETE SET NULL,
  title TEXT NOT NULL,
  status TEXT CHECK (status IN ('todo','in_progress','blocked','done','canceled')) NOT NULL DEFAULT 'todo',
  priority SMALLINT CHECK (priority BETWEEN 1 AND 5),
  due_date DATE,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Portfolio + Investments (for later phases, but useful now)
CREATE TABLE core.portfolio_company (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  portfolio_company_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_id UUID UNIQUE REFERENCES core.company(company_id) ON DELETE CASCADE,
  entry_round TEXT,
  entry_date DATE,
  board_seat BOOLEAN DEFAULT FALSE,
  ownership_pct NUMERIC(5,2),
  notes TEXT
);

CREATE TABLE core.investment (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  investment_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_id UUID REFERENCES core.company(company_id) ON DELETE CASCADE,
  amount NUMERIC(18,2),
  security_type TEXT,
  round TEXT,
  invested_on DATE,
  notes TEXT
);

-- News (TechCrunch, PitchBook, RSS)
CREATE TABLE core.news_item (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  news_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_id UUID REFERENCES core.company(company_id) ON DELETE SET NULL,
  source TEXT,               -- techcrunch, pitchbook, rss
  headline TEXT NOT NULL,
  published_at TIMESTAMPTZ,
  url TEXT,
  sentiment TEXT CHECK (sentiment IN ('positive','neutral','negative')),
  summary TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- External Integrations (tokens live in a vault; store only references)
CREATE TABLE core.integration_account (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  integration_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  provider TEXT CHECK (provider IN ('ms_graph','salesforce','monday','zoom','fathom','fireflies','granola','notion','dropbox','pitchbook','rss')),
  account_email CITEXT,
  scopes TEXT[],
  oauth_token_ref TEXT,      -- pointer into secrets manager
  status TEXT CHECK (status IN ('connected','expired','revoked')) NOT NULL DEFAULT 'connected',
  last_sync_at TIMESTAMPTZ,
  settings JSONB
);
```

---

# 3) Documents, Chunks, Embeddings (AI-native storage)

```sql
-- Raw files (pitch decks, transcripts, notes). Stored in SharePoint/Dropbox;
-- we keep the canonical URI and metadata in DB.
CREATE TABLE doc.document (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  file_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  company_id UUID REFERENCES core.company(company_id) ON DELETE SET NULL,
  deal_id UUID REFERENCES core.deal(deal_id) ON DELETE SET NULL,
  title TEXT,
  mime_type TEXT,
  location TEXT CHECK (location IN ('sharepoint','dropbox','notion','other')) NOT NULL,
  uri TEXT NOT NULL,         -- canonical URI in your storage
  checksum TEXT,             -- for de-duplication
  ingested_at TIMESTAMPTZ DEFAULT now(),
  text_extracted BOOLEAN DEFAULT FALSE
);

-- Normalize all searchable text into CHUNKS. This is the core “RAG” unit.
CREATE TABLE doc.chunk (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  chunk_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  file_id UUID REFERENCES doc.document(file_id) ON DELETE CASCADE,
  source_table TEXT,         -- e.g., 'core.interaction' when chunk comes from a meeting summary
  source_id UUID,            -- the originating row id
  ord INTEGER NOT NULL,      -- chunk order within file or source
  text TEXT NOT NULL,        -- the actual chunk content
  tsv tsvector,              -- full-text indexable column
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Embeddings live *in Postgres* via pgvector (no external vector DB needed).
-- Choose dimension to match your embedding model (example: 1536 for OpenAI text-embedding-3-small).
CREATE TABLE ai.embedding (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  chunk_id UUID PRIMARY KEY REFERENCES doc.chunk(chunk_id) ON DELETE CASCADE,
  model TEXT NOT NULL,                   -- 'text-embedding-3-small'
  dim INT NOT NULL CHECK (dim IN (768,1024,1536,3072)),
  vec VECTOR(1536) NOT NULL              -- set to chosen dim
);

-- For async embedding pipelines (workers pick jobs up and fill ai.embedding).
CREATE TABLE ai.embedding_job (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  job_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  chunk_id UUID NOT NULL REFERENCES doc.chunk(chunk_id) ON DELETE CASCADE,
  model TEXT NOT NULL,
  status TEXT CHECK (status IN ('queued','running','done','error')) NOT NULL DEFAULT 'queued',
  last_error TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

**Text search + triggers**

```sql
-- Keep tsvector up to date automatically for hybrid search
CREATE FUNCTION doc.chunk_tsv_trigger() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.tsv := to_tsvector('english', coalesce(NEW.text,''));
  RETURN NEW;
END $$;

CREATE TRIGGER trg_chunk_tsv
BEFORE INSERT OR UPDATE OF text ON doc.chunk
FOR EACH ROW EXECUTE FUNCTION doc.chunk_tsv_trigger();
```

**Indexes for speed**

```sql
-- Full-text (GIN) and trigram for fuzzy name searches
CREATE INDEX idx_company_name_trgm ON core.company USING gin (name gin_trgm_ops);
CREATE INDEX idx_chunk_tsv ON doc.chunk USING gin (tsv);
CREATE INDEX idx_doc_deal ON doc.document (deal_id);
CREATE INDEX idx_interaction_deal_time ON core.interaction (deal_id, occurred_at);

-- Vector HNSW index (fast ANN). Adjust OPTIONS to your model dimensionality & recall needs.
CREATE INDEX idx_embedding_vec_hnsw ON ai.embedding
USING hnsw (vec vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

---

# 4) Approvals, Activity Log (human-in-the-loop & audit)

```sql
CREATE TABLE ops.approval (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  approval_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  object_type TEXT,                 -- 'email','crm_record','task'
  object_table TEXT,                -- e.g., 'core.task'
  object_id UUID,                   -- row id the action is about
  action TEXT,                      -- 'send','create','update'
  proposed_payload JSONB NOT NULL,  -- diff/payload to show in Slack
  status TEXT CHECK (status IN ('pending','approved','rejected')) NOT NULL DEFAULT 'pending',
  requested_by UUID REFERENCES sec.app_user(user_id) ON DELETE SET NULL,
  decided_by UUID REFERENCES sec.app_user(user_id) ON DELETE SET NULL,
  requested_at TIMESTAMPTZ DEFAULT now(),
  decided_at TIMESTAMPTZ
);

CREATE TABLE ops.activity_log (
  tenant_id UUID NOT NULL REFERENCES sec.tenant(tenant_id) ON DELETE CASCADE,
  activity_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  actor TEXT CHECK (actor IN ('user','ai','system')) NOT NULL,
  action TEXT NOT NULL,
  object_table TEXT,
  object_id UUID,
  at TIMESTAMPTZ NOT NULL DEFAULT now(),
  metadata JSONB
);

CREATE INDEX idx_activity_time ON ops.activity_log (at DESC);
```

---

# 5) Search Helpers (pure SQL functions)

### 5.1 Semantic search (Top-K by cosine distance)

```sql
-- Return top K chunk hits by cosine distance.
-- Provide a *parameterized* embedding vector from your app layer.
CREATE OR REPLACE FUNCTION ai.search_semantic(
  p_tenant UUID,
  p_query_vec VECTOR,
  p_k INT DEFAULT 8
)
RETURNS TABLE(
  chunk_id UUID,
  file_id UUID,
  distance REAL,
  text TEXT,
  source_table TEXT,
  source_id UUID
) LANGUAGE sql STABLE AS $$
  SELECT c.chunk_id, c.file_id, (e.vec <=> p_query_vec) AS distance, c.text, c.source_table, c.source_id
  FROM ai.embedding e
  JOIN doc.chunk c ON c.chunk_id = e.chunk_id
  WHERE e.tenant_id = p_tenant
  ORDER BY e.vec <=> p_query_vec
  LIMIT p_k;
$$;
```

### 5.2 Full-text search

```sql
CREATE OR REPLACE FUNCTION ai.search_keyword(
  p_tenant UUID,
  p_query TEXT,
  p_k INT DEFAULT 20
)
RETURNS TABLE(
  chunk_id UUID,
  file_id UUID,
  rank REAL,
  text TEXT,
  source_table TEXT,
  source_id UUID
) LANGUAGE sql STABLE AS $$
  SELECT c.chunk_id, c.file_id,
         ts_rank(c.tsv, plainto_tsquery('english', p_query)) AS rank,
         c.text, c.source_table, c.source_id
  FROM doc.chunk c
  WHERE c.tenant_id = p_tenant
    AND c.tsv @@ plainto_tsquery('english', p_query)
  ORDER BY rank DESC
  LIMIT p_k;
$$;
```

### 5.3 Hybrid search (semantic + keyword)

```sql
-- alpha in [0..1]: 0 = keyword only, 1 = semantic only
CREATE OR REPLACE FUNCTION ai.search_hybrid(
  p_tenant UUID,
  p_query TEXT,
  p_query_vec VECTOR,
  p_k INT DEFAULT 12,
  p_alpha REAL DEFAULT 0.5
)
RETURNS TABLE(
  chunk_id UUID,
  file_id UUID,
  hybrid_score REAL,
  text TEXT,
  source_table TEXT,
  source_id UUID
) LANGUAGE sql STABLE AS $$
  WITH kw AS (
    SELECT c.chunk_id,
           ts_rank(c.tsv, plainto_tsquery('english', p_query)) AS kw_score
    FROM doc.chunk c
    WHERE c.tenant_id = p_tenant
  ),
  sem AS (
    SELECT e.chunk_id,
           1.0 - (e.vec <=> p_query_vec) AS sem_score  -- convert distance to similarity
    FROM ai.embedding e
    WHERE e.tenant_id = p_tenant
  )
  SELECT c.chunk_id, c.file_id,
         (coalesce(kw.kw_score, 0) * (1 - p_alpha) + coalesce(sem.sem_score, 0) * p_alpha) AS hybrid_score,
         c.text, c.source_table, c.source_id
  FROM doc.chunk c
  LEFT JOIN kw ON kw.chunk_id = c.chunk_id
  LEFT JOIN sem ON sem.chunk_id = c.chunk_id
  WHERE c.tenant_id = p_tenant
  ORDER BY hybrid_score DESC
  LIMIT p_k;
$$;
```

---

# 6) Ingestion & Embedding Workflow

### 6.1 Insert a document + chunks

```sql
-- 1) Add the document row (SharePoint/Dropbox/Notion link)
INSERT INTO doc.document (tenant_id, company_id, deal_id, title, mime_type, location, uri, checksum)
VALUES ($tenant, $company_id, $deal_id, 'Pitch Deck – Orion Mobility', 'application/pdf', 'sharepoint',
        'https://sharepoint.example/...', 'sha256:...');

-- 2) For each extracted chunk:
INSERT INTO doc.chunk (tenant_id, file_id, source_table, source_id, ord, text)
VALUES ($tenant, $file_id, 'doc.document', $file_id, 1, $chunk_text_1),
       ($tenant, $file_id, 'doc.document', $file_id, 2, $chunk_text_2), ...;

-- 3) Queue embedding jobs (async worker will fill ai.embedding.vec)
INSERT INTO ai.embedding_job (tenant_id, chunk_id, model)
SELECT $tenant, chunk_id, 'text-embedding-3-small'
FROM doc.chunk WHERE file_id = $file_id;
```

### 6.2 Worker fills embeddings

```sql
-- Pseudocode outline executed by your worker:
-- 1) SELECT * FROM ai.embedding_job WHERE status='queued' LIMIT 100 FOR UPDATE SKIP LOCKED;
-- 2) Call your embedding API on each chunk.text
-- 3) UPSERT into ai.embedding
INSERT INTO ai.embedding (tenant_id, chunk_id, model, dim, vec)
VALUES ($tenant, $chunk_id, 'text-embedding-3-small', 1536, $vector)
ON CONFLICT (chunk_id) DO UPDATE SET vec = EXCLUDED.vec, model = EXCLUDED.model, dim = EXCLUDED.dim;

UPDATE ai.embedding_job SET status='done', updated_at=now() WHERE job_id=$job;
```

---

# 7) Example Queries You’ll Actually Use

### 7.1 “Give me context on ACME Mobility before my call”

```sql
-- Get the company, latest deal, recent interactions, key docs
SELECT c.name, c.sector, c.stage, d.status, d.thesis_tags
FROM core.company c
LEFT JOIN core.deal d ON d.company_id = c.company_id
WHERE c.tenant_id = $tenant AND c.name ILIKE '%acme mobility%';

SELECT occurred_at, type, summary
FROM core.interaction i
WHERE i.tenant_id=$tenant AND i.deal_id=$deal_id
ORDER BY occurred_at DESC
LIMIT 5;

SELECT title, uri
FROM doc.document
WHERE tenant_id=$tenant AND deal_id=$deal_id
ORDER BY ingested_at DESC
LIMIT 5;
```

### 7.2 Hybrid search over your corpus

```sql
-- Pass (p_query, p_query_vec) from your Slack command handler
SELECT * FROM ai.search_hybrid($tenant, 'Detroit mobility charging hardware', $vec, 12, 0.6);
```

### 7.3 “What did I discuss with Founder X last month?”

```sql
SELECT i.occurred_at, i.summary
FROM core.interaction i
JOIN core.person p ON p.person_id = i.person_id
WHERE i.tenant_id=$tenant
  AND p.full_name ILIKE '%Founder X%'
  AND i.occurred_at >= date_trunc('month', now()) - INTERVAL '1 month'
  AND i.type IN ('meeting','call')
ORDER BY i.occurred_at DESC;
```

### 7.4 Approve an action proposed by the AI

```sql
-- View pending approvals
SELECT approval_id, object_type, action, proposed_payload, requested_at
FROM ops.approval
WHERE tenant_id=$tenant AND status='pending'
ORDER BY requested_at ASC;

-- Approve
UPDATE ops.approval
SET status='approved', decided_by=$user_id, decided_at=now()
WHERE tenant_id=$tenant AND approval_id=$approval_id;
```

---

# 8) Performance Notes

* **pgvector HNSW** is fast for Top-K semantic search. Tune `m` and `ef_construction` for recall/space.
* Keep chunks ~500–1000 tokens. Store original text in `doc.chunk.text` for exact quotes; use `doc.document.uri` to open source.
* Create **partial indexes** by tenant if the corpus grows large.
* Consider **compression** (`TOAST`) for big `text` but avoid overly giant single rows—chunking solves this.

---

# 9) Security (RLS-ready)

If/when you invite other fund members, enable **Row-Level Security** and bind app sessions to a `tenant_id`:

```sql
ALTER TABLE core.company ENABLE ROW LEVEL SECURITY;
-- Repeat for all tenant tables…

CREATE POLICY tenant_isolation ON core.company
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- In your connection pooler/app, SET app.tenant_id = '<tenant-uuid>' on session.
```

---

# 10) What this “zero-DB AI-native” design gives you

* **One database** does it all: **entities + full-text + vectors** (no separate vector store).
* **Hybrid retrieval** out of the box (semantic + keyword).
* **Auditable** approvals + actions (crucial for investor workflows).
* **Incremental ingestion** via `doc.document` → `doc.chunk` → `ai.embedding_job`.
* **Future-proof** multi-tenant and RLS when you add teammates.

---
