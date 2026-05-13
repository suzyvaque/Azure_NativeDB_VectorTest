# Plan: Azure Vector DB Test Framework

**TL;DR:** Build a structured test suite in this repo that benchmarks vector similarity search on Azure Cosmos DB for NoSQL and Azure Database for PostgreSQL Flexible Server (pgvector). Uses real ANN Benchmarks data, covers all supported index types per database, and scales from basic recall/latency tests up to high-QPS throughput. Python throughout, dual auth strategy (.env local / Managed Identity CI).

---

## Decisions & Scope

- **Vector dimensions / distance metric:** 1536-dim / cosine (OpenAI text-embedding-ada-002)
- **Dataset:** Real ANN Benchmarks HDF5 — `dbpedia-openai-1536-angular` (1M × 1536-dim cosine); `sift-128-euclidean` (128-dim) for Cosmos DB flat index
- **PostgreSQL target:** Azure Database for PostgreSQL Flexible Server (with pgvector + pg_diskann)
- **ResourcePrep format:** Azure CLI scripts now; Bicep templates noted as future work
- **Caching:** Phase 8 is a stub/placeholder only
- **Auth:** `.env` file locally, Managed Identity in CI
- **Azure Cosmos DB for PostgreSQL excluded** — it is on a retirement path
- **Flat index gets a separate 128-dim dataset** — Cosmos DB flat index caps at 505 dims, incompatible with 1536-dim; `vec-flat` container uses 128-dim vectors
- **One container per Cosmos DB index type** — vector policies are immutable post-creation; containers named `vec-flat`, `vec-qflat`, `vec-qflat-spherical`, `vec-diskann`

---

## Repo Structure (target)

```
Azure_NativeDB_VectorTest/
├── README.md
├── requirements.txt
├── .env.template
├── Plan/
│   └── plan-azureNativeDbVectorTest.prompt.md   ← this file
├── ResourcePrep/
│   ├── README.md
│   ├── 01_create_rg_cosmos.sh
│   ├── 02_create_postgresql.sh
│   └── assign_managed_identity.sh
├── DDL/
│   ├── README.md
│   ├── cosmos/
│   │   ├── container_policy_flat.json
│   │   ├── container_policy_quantizedFlat.json
│   │   ├── container_policy_quantizedFlat_spherical.json
│   │   └── container_policy_diskANN.json
│   └── postgresql/
│       ├── 01_enable_extensions.sql
│       ├── 02_create_table.sql
│       └── 03_create_indexes.sql
├── AzureCosmosDB/
│   ├── setup_containers.py
│   ├── load_data.py
│   ├── test_basic.py
│   ├── test_throughput.py
│   ├── utils/
│   │   ├── dataset.py
│   │   └── metrics.py
│   └── results/
└── AzurePostgreSQL/
    ├── setup_db.py
    ├── load_data.py
    ├── test_basic.py
    ├── test_throughput.py
    ├── utils/
    │   └── metrics.py
    └── results/
```

---

## Phase 0 — Repo Foundation

1. **`requirements.txt`** — `azure-cosmos`, `azure-identity`, `psycopg[binary]`, `pgvector`, `numpy`, `h5py`, `pandas`, `tqdm`, `python-dotenv`
2. **`.env.template`** — Placeholders for `COSMOS_ENDPOINT`, `COSMOS_KEY`, `COSMOS_DATABASE`, `PG_HOST`, `PG_USER`, `PG_PASSWORD`, `PG_DATABASE`; `USE_MANAGED_IDENTITY=false`
3. **`ResourcePrep/01_create_rg_cosmos.sh`** — `az group create` + `az cosmosdb create` with `--capabilities EnableNoSQLVectorSearch`; echos endpoint/key for `.env`
4. **`ResourcePrep/02_create_postgresql.sh`** — `az postgres flexible-server create` (Burstable B2ms tier); `az postgres flexible-server parameter set` to allowlist `vector,pg_diskann` in `azure.extensions`; firewall rule for local IP
5. **`ResourcePrep/assign_managed_identity.sh`** — Assigns Cosmos DB built-in data contributor role + PostgreSQL AAD admin, for CI pipelines
6. **`README.md`** — Project overview, prerequisites, quick-start instructions

---

## Phase 1 — DDL

7. **`DDL/cosmos/container_policy_flat.json`** — 128-dim, cosine, `"type": "flat"`; embedding path `/embedding`
   > ⚠️ Flat index maximum is **505 dimensions**. Uses 128-dim SIFT dataset, not the 1536-dim primary dataset.

8. **`DDL/cosmos/container_policy_quantizedFlat.json`** — 1536-dim, cosine, `"type": "quantizedFlat"`, `quantizerType: "product"`, `quantizationByteSize: 64`

9. **`DDL/cosmos/container_policy_quantizedFlat_spherical.json`** — Same as above but `quantizerType: "spherical"` (public preview)

10. **`DDL/cosmos/container_policy_diskANN.json`** — 1536-dim, cosine, `"type": "diskANN"`, `indexingSearchListSize: 100`

11. **`DDL/postgresql/01_enable_extensions.sql`**
    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    CREATE EXTENSION IF NOT EXISTS pg_diskann CASCADE;
    ```

12. **`DDL/postgresql/02_create_table.sql`**
    ```sql
    CREATE TABLE IF NOT EXISTS items (
        id        BIGSERIAL PRIMARY KEY,
        doc_id    TEXT NOT NULL,
        content   TEXT,
        embedding VECTOR(1536)
    );
    ```

13. **`DDL/postgresql/03_create_indexes.sql`** — Commented blocks per index type; applied selectively per test run:
    - `ivfflat (embedding vector_cosine_ops) WITH (lists = 100)`
    - `hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64)`
    - `diskann (embedding vector_cosine_ops) WITH (max_neighbors = 32)` (pg_diskann)

---

## Phase 2 — Shared Utilities *(parallel with Phase 1)*

14. **`utils/dataset.py`**
    - Downloads ANN Benchmarks HDF5 from `http://ann-benchmarks.com/` and caches locally
    - Primary: `dbpedia-openai-1536-angular.hdf5` → `train`, `test`, `neighbors`, `distances`
    - Flat-index: `sift-128-euclidean.hdf5` → 128-dim equivalent
    - Exposes `get_dataset(name, max_vectors=None) -> DatasetResult`
    - Supports `--num-vectors` cap for small test runs without full download

15. **`utils/metrics.py`**
    - `recall_at_k(predicted_ids, true_ids, k) -> float`
    - `latency_stats(timings_sec: list) -> dict` with `mean`, `p50`, `p99`
    - `save_results(name, index_type, n_vectors, k, metrics, out_dir)`

---

## Phase 3 — Cosmos DB Setup & Load

16. **`AzureCosmosDB/setup_containers.py`**
    - Reads all 4 `DDL/cosmos/*.json` files
    - Calls `database.create_container_if_not_exists(id, partition_key, indexing_policy, vector_embedding_policy)` for each
    - Idempotent; prints container IDs on success
    - Auth: `azure-identity` `DefaultAzureCredential` if `USE_MANAGED_IDENTITY=true`, else key from `.env`

17. **`AzureCosmosDB/load_data.py`**
    - CLI args: `--container`, `--num-vectors` (default 5000), `--batch-size` (default 100)
    - Loads dataset via `utils/dataset.py`
    - Calls `container.upsert_item()` in batches; `tqdm` progress bar
    - Stores `{"id": str(i), "doc_id": ..., "embedding": [...]}` documents

---

## Phase 4 — Cosmos DB Basic Tests

18. **`AzureCosmosDB/test_basic.py`**
    - Iterates over applicable containers (flat → qflat → qflat-spherical → diskANN)
    - Runs 100 queries each using:
      ```sql
      SELECT TOP 10 c.doc_id,
             VectorDistance(c.embedding, @q) AS score
      FROM c
      ORDER BY VectorDistance(c.embedding, @q)
      ```
      with `enable_cross_partition_query=True`
    - Computes `recall@10` vs ANN Benchmarks ground truth neighbors
    - Records per-query latency
    - Outputs JSON to `AzureCosmosDB/results/basic_{container}_{timestamp}.json`

---

## Phase 5 — PostgreSQL Setup & Load

19. **`AzurePostgreSQL/setup_db.py`**
    - Connects via `psycopg.connect()` (psycopg3) with SSL; runs DDL files 01 and 02 in order
    - Calls `register_vector(conn)` from `pgvector.psycopg`
    - Azure portal prerequisite: enable `vector` and `pg_diskann` in `azure.extensions` server parameter

20. **`AzurePostgreSQL/load_data.py`**
    - CLI args: `--num-vectors`, `--batch-size`
    - Uses `executemany()` with numpy array embeddings via `pgvector` type adapter
    - `tqdm` progress bar

---

## Phase 6 — PostgreSQL Basic Tests

21. **`AzurePostgreSQL/test_basic.py`**
    - Iterates over index types: `none (seq scan)` → `ivfflat` → `hnsw` → `diskann`
    - Per iteration: `DROP INDEX IF EXISTS vec_idx; CREATE INDEX vec_idx ...`; `SET enable_seqscan = OFF`
    - Runs 100 queries: `SELECT doc_id FROM items ORDER BY embedding <=> %s LIMIT 10`
    - Computes `recall@10` + latency stats
    - Outputs JSON to `AzurePostgreSQL/results/basic_{index}_{timestamp}.json`

---

## Phase 7 — High Throughput

22. **`AzureCosmosDB/test_throughput.py`**
    - Uses `azure.cosmos.aio.CosmosClient` (async)
    - Concurrency sweep: 1 / 8 / 32 / 64 simultaneous queries via `asyncio.gather()`
    - Targets the `vec-diskann` container (best throughput candidate)
    - Measures sustained QPS and p50/p99 latency over a 60-second window
    - Outputs `AzureCosmosDB/results/throughput_diskann_{timestamp}.json`

23. **`AzurePostgreSQL/test_throughput.py`**
    - Uses `asyncpg` connection pool with `register_vector` from `pgvector.asyncpg`
    - Same concurrency sweep: 1 / 8 / 32 / 64
    - Targets HNSW index (best throughput candidate for pgvector)
    - Same 60-second sustained window
    - Outputs `AzurePostgreSQL/results/throughput_hnsw_{timestamp}.json`

---

## Phase 8 — Caching (stub)

24. Placeholder `AzureCosmosDB/caching/README.md` and `AzurePostgreSQL/caching/README.md` with:
    - Commented architecture: Redis semantic cache (Azure Cache for Redis) → check cache before DB query → store result with TTL
    - Notes on exact-match vs. approximate semantic cache key strategies
    - No implementation yet; revisit after throughput results

---

## Key Technical Constraints (reference)

| Constraint | Detail |
|---|---|
| Cosmos DB flat index max dims | **505** — use 128-dim dataset |
| Cosmos DB quantizedFlat/diskANN max dims | 4,096 — 1536-dim ✓ |
| Min vectors for quantizedFlat/diskANN index | ≥ 1,000 — below this Cosmos DB falls back to full scan |
| Vector policy immutable | Cannot be changed post-container-creation; must drop + recreate |
| pgvector version on Azure | 0.8.2 (PostgreSQL 13–18) |
| pg_diskann | Azure-specific preview extension; requires allowlisting in portal |
| Cosmos distance functions in query | `VectorDistance()` in `SELECT` + `ORDER BY`; always use `TOP N` |
| PostgreSQL distance operators | `<->` Euclidean, `<#>` inner product, `<=>` cosine |
| Cosmos DB shared throughput | Vector search NOT supported — use dedicated RU/s |

---

## Verification Checklist

- [ ] `pip install -r requirements.txt` — clean install, no conflicts
- [ ] `bash ResourcePrep/01_create_rg_cosmos.sh` → Cosmos DB account in portal with vector capability
- [ ] `bash ResourcePrep/02_create_postgresql.sh` → PostgreSQL server with `vector`, `pg_diskann` in `azure.extensions`
- [ ] `python AzureCosmosDB/setup_containers.py` → 4 containers visible in Data Explorer with vector indexes
- [ ] `python AzureCosmosDB/load_data.py --container vec-diskann --num-vectors 5000` → loads without RU errors
- [ ] `python AzureCosmosDB/test_basic.py` → prints recall@10 + p99 latency table for all 4 index types
- [ ] `python AzurePostgreSQL/setup_db.py` → table + extensions visible; no errors
- [ ] `python AzurePostgreSQL/load_data.py --num-vectors 5000` → loads without errors
- [ ] `python AzurePostgreSQL/test_basic.py` → prints recall@10 + latency for none/ivfflat/hnsw/diskann
- [ ] Both throughput scripts produce a QPS-vs-concurrency table

---

## Open Items

- [ ] **Dataset download strategy:** The `dbpedia-openai-1536-angular` HDF5 is ~5–6 GB. Decide: full download + cache, or streaming partial read of first N rows?
- [ ] **Bicep templates:** Deferred — add to `ResourcePrep/` after CLI scripts are validated
- [ ] **Caching implementation:** Revisit after Phase 7 throughput results; Redis vs. in-process TBD
- [ ] **Result visualization:** Add a notebook or plotting script to compare index types across DBs
