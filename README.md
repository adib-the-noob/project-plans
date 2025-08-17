<img width="2535" height="1418" alt="image" src="https://github.com/user-attachments/assets/d6fda7f1-1f2f-4591-8aed-fd5349984177" />

# Systems Trilogy: Build-Your-Own Backend Stack

A hands-on, three-project journey to master networking, distributed systems, databases, and systems design by **building**:

1. **`microfast`** – a tiny FastAPI-like web framework from scratch (networking + HTTP internals)
2. **`unamex`** – a 1M-scale username search & suggestion engine (multi-region, sharding, Redis, Bloom filter, Airflow)
3. **`tinyorm`** – a minimal but production-grade ORM (query builder, migrations, unit-of-work)

> **Why this repo?** Because you learn real systems by *building* them, end-to-end.

---

## TL;DR

* **You’ll learn:** sockets, HTTP/1.1 & ASGI, event loops, async I/O, schedulers, sharding, replication, caching, probabilistic data structures, eventual consistency, job orchestration, query planning, connection pooling, migrations, and API ergonomics.
* **You’ll produce:** a mini web framework, a horizontally scalable search service for 1M+ usernames with suggestions, and your own ORM.

---

## What These Projects Teach You

| Topic Area              | microfast (Framework)                                         | unamex (Username Search)                          | tinyorm (ORM)                      |
| ----------------------- | ------------------------------------------------------------- | ------------------------------------------------- | ---------------------------------- |
| **Networking & OS**     | TCP sockets, epoll/kqueue, Nagle’s, backlog, zero-copy basics | gRPC/HTTP, global LB, health checks, keep-alive   | connection pools, timeouts         |
| **Protocols**           | HTTP/1.1 parsing, chunked, keep-alive, WebSocket              | REST/gRPC, pagination, rate-limit headers         | SQL dialect nuance, prepared stmts |
| **Concurrency**         | event loop, coroutines, async/await, task scheduling          | workers, fan-out/fan-in, idempotency              | transactions, unit-of-work         |
| **Performance**         | routing tries, middleware cost, context switches              | cache hit ratios, Bloom FPR, p95 latency          | batch inserts, lazy/eager loading  |
| **Distributed Systems** | ASGI, backpressure, graceful shutdown                         | sharding, replication, multi-region failover      | optimistic/pessimistic locking     |
| **Data Engineering**    | –                                                             | Airflow DAGs, ingestion pipelines, synthetic data | migrations, schema evolution       |
| **Search/Suggestion**   | –                                                             | exact match, prefix, embeddings/phonetics         | query builder optimization         |

---

## Monorepo Structure

```
.
├── microfast/         # Project 1 – mini FastAPI-like framework
│   ├── microfast/     # core lib
│   ├── examples/
│   ├── benchmarks/
│   └── tests/
├── unamex/            # Project 2 – username search engine
│   ├── api/           # stateless API service
│   ├── ingest/        # Airflow DAGs, producers
│   ├── workers/       # batch & stream processors
│   ├── infra/         # docker, terraform, k8s manifests
│   └── tests/
├── tinyorm/           # Project 3 – simple ORM
│   ├── tinyorm/
│   ├── migrations/
│   └── tests/
├── docs/              # notes, diagrams, decisions (ADR)
└── Makefile           # common dev tasks
```

---

# Project 1 — `microfast`: A Tiny FastAPI-like Framework

### Goal

Build a minimal, high-performance web framework that teaches **network stacks, HTTP parsing, ASGI, and async concurrency**.

### Core Concepts

* Raw TCP **sockets** (blocking → non-blocking → epoll/kqueue)
* **HTTP/1.1** parser (request line, headers, chunked body, keep-alive)
* **Router** (path → handler, params, tries, middleware pipeline)
* **ASGI-like** interface (request/response lifecycle, background tasks)
* **Concurrency** (event loop, task queue, cancellation, graceful shutdown)
* **WebSocket** upgrade path (handshake, frames) – stretch goal

### Milestones

1. Echo server → hello world HTTP → parse method/path/headers
2. Keep-alive + connection reuse + timeouts
3. Router: static, params, wildcard; middleware chain
4. Request/Response objects; body streaming; JSON/form
5. Async handlers, background tasks, context-local storage
6. Static files, CORS, compression (gz/deflate), logging
7. Benchmarks vs Uvicorn/FastAPI (wrk, autocannon)
8. Stretch: HTTP/2 (h2), WebSocket, TLS, zero-copy sendfile

### Example (sketch)

```py
app = MicroFast()

@app.get("/users/{id}")
async def get_user(req, id: int):
    return {"id": id}

app.run(host="0.0.0.0", port=8080)
```

### Benchmarks

* Targets: **<1ms** in-process handler, **≥10k rps** on laptop with hello route

---

# Project 2 — `unamex`: 1M Username Search & Suggestion (Multi-Region)

### Goal

A globally distributed API to **check username availability**, power **prefix search**, and provide **suggestions** with **1M+ entries**, **Redis caching**, **Bloom filter**, and **DB sharding**.

### High-Level Architecture

```
[Client]
   │
[Global Anycast/GeoDNS]
   │
[Region A API]──┬──[Redis A]──[Bloom A]
   │            │
   │            └──[Shard DBs A1..An]
   │
[Region B API]──┬──[Redis B]──[Bloom B]
                │
                └──[Shard DBs B1..Bn]

Ingest: [Airflow] → [Kafka/SQS] → [Workers] → [Sharded DB]
```

### Key Components

* **Synthetic Data Generation**: LLM prompts + rules → usernames (10⁶)
* **Airflow DAGs**: generate → validate/normalize → dedup → shard-assign → load
* **Shard Strategy**: hash(username) → consistent-hash ring → shard N
* **Bloom Filter**: fast negative checks (tunable false-positive rate, e.g., 1%)
* **Redis**: cache hot lookups + prefix indices; TTL-based invalidation
* **API**: check availability; prefix search; suggestions; rate limiting
* **Multi-Region**: active-active; per-region Redis/Bloom; async cross-region replication

### Data Model (Postgres example)

```sql
CREATE TABLE usernames (
  id BIGSERIAL PRIMARY KEY,
  username TEXT NOT NULL,
  norm_username TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON usernames (norm_username);
-- Optional: prefix index via trigram or dedicated table
```

### Normalization Rules

* lowercase, NFC Unicode normalize, trim, collapse underscores/dots per policy
* reserved prefixes/suffixes blacklist (e.g., admin, support)

### Bloom + DB Check (pseudocode)

```py
norm = normalize(u)
if not bloom.might_contain(norm):
    return AVAILABLE
if redis.get(norm) == TAKEN:
    return TAKEN
# shard lookup
db = shard_for(norm)
exists = db.exists("SELECT 1 FROM usernames WHERE norm_username = $1", norm)
redis.set(norm, TAKEN if exists else AVAILABLE, ex=3600)
return TAKEN if exists else AVAILABLE
```

### Suggestions

* **Heuristics**: append year, separators, leetspeak, synonyms
* **Phonetic**: Soundex/Metaphone for romanized inputs
* **Vector** (optional): char n-gram embeddings; ANN (FAISS) for similar handles
* **Prefix Search**: Redis sorted sets or Radix/Trie in memory (sync from DB)

### Airflow DAG (sketch)

```
@generation → @normalize → @dedup → @partition → @load_shards → @build_bloom → @refresh_caches
```

### Sharding Options

* **Hash sharding**: uniform; poor range queries
* **Range sharding**: great for prefixes; needs rebalancing
* **Consistent hashing**: easy scale-out; stable key mapping

### Multi-Region Notes

* Region-local writes; periodic async replication to peers
* Conflict policy: **last-writer-wins** on norm\_username or strong single-writer via queue
* Health checks, circuit breakers, and brownout mode if a region is degraded

### SLOs & Metrics

* **SLO**: p95 ≤ 30ms for cache hits; ≤ 150ms for DB hits
* **Dashboards**: hit ratio, FPR, shard skew, replication lag, rate limit drops

### Local Dev (docker-compose)

* Postgres (N shards), Redis, API, Airflow, Worker
* Makefile targets: `make up`, `make seed`, `make bench`

---

# Project 3 — `tinyorm`: Build Your Own ORM

### Goals

* Ergonomic API over DB drivers with **prepared statements**, **connection pooling**, **transactions**, and **migrations**.
* Query builder that emits dialect-correct SQL; optional **schema-first** models.

### Core Concepts

* **Connection Pool**: max size, queue, timeouts, health checks
* **Query Builder**: AST → SQL string + bound params
* **Migrations**: version table, up/down, autogenerate diffs (stretch)
* **Unit of Work**: track dirty entities, commit/rollback
* **Eager/Lazy Loading**: N+1 controls; includes/join strategy

### Minimal API (sketch)

```py
from tinyorm import Model, fields, DB

DB.configure(url="postgres://...")

class User(Model):
    id = fields.Int(pk=True)
    username = fields.Text(unique=True)
    created_at = fields.Timestamp(default="now()")

async with DB.transaction():
    u = await User.create(username="adib")
    u.username = "adib_dev"
    await u.save()

users = await User.select().where(User.username.startswith("a")).limit(10).all()
```

### Milestones

1. Sync driver over psycopg/asyncpg; basic CRUD
2. Query builder with `select/where/order/limit`
3. Connection pool + transactions + retries (idempotency keys)
4. Migrations CLI (`tinyorm migrate`, `tinyorm makemigrations`)
5. Relationships: FK, 1\:N, M\:N; eager/lazy
6. Integrations: `microfast` plugin, `unamex` models

---

## Getting Started

### Prerequisites

* Python 3.11+
* Docker + Docker Compose
* Make, GNU coreutils

### Quickstart

```
make bootstrap        # create virtualenvs, pre-commit hooks
make up               # start local stack (compose)
make seed             # generate synthetic usernames via script
make bench            # run basic benchmarks
```

### Environment

```
cp .env.example .env
# edit DB URLs, Redis, shard counts, region id
```

---

## Development Guide

* **CI**: lint (ruff), typecheck (mypy), tests (pytest)
* **Observability**: logs (JSON), metrics (Prometheus), traces (OTel)
* **Docs**: `docs/` with ADRs (Architecture Decision Records)
* **Security**: input normalization, rate limiting, circuit breakers
* **Reliability**: readiness/liveness probes, graceful shutdown, backpressure

---

## Benchmarks (Targets)

* `microfast`: 10–50k rps hello-route on laptop; p99 < 3ms
* `unamex`: cache hit p95 < 30ms; DB hit p95 < 150ms; FPR ≤ 1%
* `tinyorm`: 100k insert/min (batch), single query p95 < 5ms (hot cache)

---

## Roadmap (suggested 4–6 weeks)

**Week 1**: microfast – socket → HTTP → router → middleware → async

**Week 2**: microfast – ASGI compat, streaming, gzip, CORS, basic bench

**Week 3**: unamex – data generation (LLM), Airflow ingest, sharding, Redis, Bloom

**Week 4**: unamex – API, prefix search, suggestions, dashboards, SLOs

**Week 5**: tinyorm – CRUD, pool, transactions, query builder, tests

**Week 6**: tinyorm – migrations, relationships, microfast integration, polish

---

## Make Targets (examples)

```
make up          # docker compose up -d
make down        # docker compose down -v
make seed        # generate usernames, load shards
make bench       # run wrk/autocannon load test scripts
make test        # pytest -q
make lint        # ruff check
make type        # mypy .
```

---

## License

MIT (see LICENSE)

## Author

Adib – Systems builder in training.
