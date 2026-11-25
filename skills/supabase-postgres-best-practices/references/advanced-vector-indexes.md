---
title: Choose and Tune pgvector Indexes for Reliable Filtered Search
impact: HIGH
impactDescription: Improves vector-search recall and prevents filtered queries from silently returning too few results
tags: pgvector, vector-search, hnsw, ivfflat, rag, indexes
---

## Choose and Tune pgvector Indexes for Reliable Filtered Search

Approximate vector indexes trade build time, memory, speed, and recall. Choose HNSW when query quality and latency matter most, and choose IVFFlat when lower build cost and memory usage are more important. Always test the index with the same filters and embedding model used in production.

**Incorrect (create an approximate index without considering query filters):**

```sql
create extension if not exists vector with schema extensions;

create table documents (
  id bigint generated always as identity primary key,
  tenant_id uuid not null,
  embedding extensions.vector(1536) not null
);

create index documents_embedding_idx
  on documents using hnsw (embedding vector_cosine_ops);

-- Filtering happens after the approximate index scan. With the default
-- search budget, this can return fewer than 5 matching rows even when
-- at least 5 rows exist for the tenant.
select id
from documents
where tenant_id = '00000000-0000-0000-0000-000000000001'
order by embedding <=> '[0.1, 0.2, 0.3]'::extensions.vector
limit 5;
```

**Correct (index the filter and use iterative scans when approximate search is filtered):**

```sql
-- Start with a normal index for the filter column. This can be effective
-- when the filter selects a small fraction of the table.
create index documents_tenant_id_idx on documents (tenant_id);

-- For pgvector versions that support iterative scans, let the HNSW scan
-- continue until it finds enough qualifying rows. Scope the setting to the
-- transaction so it does not change unrelated queries.
begin;
set local hnsw.iterative_scan = strict_order;
set local hnsw.max_scan_tuples = 20000;

select id
from documents
where tenant_id = '00000000-0000-0000-0000-000000000001'
order by embedding <=> '[0.1, 0.2, 0.3]'::extensions.vector
limit 5;

commit;
```

For a new dataset, HNSW is usually the practical starting point because it provides a strong speed/recall tradeoff without requiring a training phase. Tune `ef_construction` only when index-build time and recall measurements justify it. IVFFlat can use less memory and build faster, but it should be created after the table contains representative data; choose `lists` and `ivfflat.probes` from measurements rather than copying a default blindly.

When tenants have very different data distributions or strict isolation requirements, consider partitioning by tenant instead of relying on one shared approximate index. For a small, highly selective filter, a regular B-tree filter index may be enough for exact nearest-neighbor search. For a broad filter, iterative approximate scans or partitioning may be more appropriate.

Reference: [pgvector indexing and filtering](https://github.com/pgvector/pgvector#ivfflat)
Reference: [Supabase pgvector guide](https://supabase.com/docs/guides/database/extensions/pgvector)
