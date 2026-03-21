# nlspec Server — Layer 3: Configuration

> **Version:** 1.0.0
> **Author:** Divyendu Deepak Singh
> **Date:** March 2026
> **Type:** COMPONENT
> **Language Target:** Python
> **Status:** Draft
> **Layer:** 3-Configuration
> **DERIVES FROM:** L2-nlspec-workflows-spec

---

## Table of Contents

1. [Configuration](#configuration)
2. [Deployment Artifacts](#deployment-artifacts)
3. [File Structure](#filestructure)
4. [Build and Run](#build-and-run)
5. [Dependencies](#dependencies)

---

## Configuration

All configuration is environment-variable driven. Defaults allow local dev without any setup.

### CONFIG: server

```
CONFIG server.transport:
  type: String
  default: "stdio"
  env: NLSPEC_TRANSPORT
  description: MCP transport type. "stdio" (Phase 1, agent spawns server) or "sse" (Phase 2, server listens on port)
  validation: "stdio" | "sse"

CONFIG server.project_dir:
  type: String
  default: "."
  env: NLSPEC_PROJECT_DIR
  description: Root directory of the project
  validation: must exist

CONFIG server.specs_dir:
  type: String
  default: "."
  env: NLSPEC_SPECS_DIR
  description: Root directory of specs/ folder. Relative paths are relative to project_dir.

CONFIG server.grpc_port:
  type: u16
  default: 50060
  env: NLSPEC_GRPC_PORT
  description: Port to bind gRPC server to (Phase 2+)

CONFIG server.host:
  type: String
  default: "127.0.0.1"
  env: NLSPEC_HOST
  description: Host to bind SSE server to (only used if transport="sse")

CONFIG server.port:
  type: u16
  default: 8080
  env: NLSPEC_PORT
  description: Port to bind SSE server to (only used if transport="sse")
```

### CONFIG: database

```
CONFIG database.type:
  type: String
  default: "sqlite" (Phase 1) or "postgresql" (Phase 2)
  env: NLSPEC_DATABASE_TYPE
  description: Database backend for indexing.
  validation: "postgresql" | "sqlite"

CONFIG database.url:
  type: String | None
  default: None
  env: DATABASE_URL
  description: PostgreSQL connection string (e.g., postgresql://user:pass@localhost:5432/nlspec)
               If not set and database.type="sqlite", uses .nlspec/index.sqlite

CONFIG database.sqlite_path:
  type: String
  default: ".nlspec/index.sqlite"
  env: NLSPEC_SQLITE_PATH
  description: SQLite index path relative to project_dir

CONFIG database.auto_reindex:
  type: bool
  default: true
  env: NLSPEC_AUTO_REINDEX
  description: Re-parse markdown files when they change (via file watcher)

CONFIG database.pool_size:
  type: u16
  default: 5 (Phase 1) or 10 (Phase 2)
  env: NLSPEC_DB_POOL_SIZE
  description: PostgreSQL connection pool size (only used if type="postgresql")
```

### CONFIG: search

```
CONFIG search.fts_enabled:
  type: bool
  default: true
  env: NLSPEC_FTS
  description: Enable full-text search. If false, only structural (reference) search works.

CONFIG search.fts_backend:
  type: String
  default: "sqlite" (Phase 1) or "postgresql" (Phase 2)
  env: NLSPEC_FTS_BACKEND
  description: FTS backend. "sqlite" uses FTS5, "postgresql" uses tsvector/tsquery + pg_trgm.
  validation: "sqlite" | "postgresql"

CONFIG search.min_relevance:
  type: float
  default: 0.1
  env: NLSPEC_MIN_RELEVANCE
  description: Minimum relevance score (0.0-1.0) for search results. Results below this are filtered.
```

### CONFIG: namespaces (Phase 2)

```
CONFIG namespaces.default:
  type: String
  default: "default"
  env: NLSPEC_DEFAULT_NAMESPACE
  description: Namespace used when no namespace is specified in tool calls

CONFIG namespaces.auto_create:
  type: bool
  default: true
  env: NLSPEC_AUTO_CREATE_NAMESPACE
  description: If true, nlspec_import creates namespaces automatically
```

### CONFIG: patches (Phase 2)

```
CONFIG patches.dir:
  type: String
  default: "patches/"
  env: NLSPEC_PATCHES_DIR
  description: Directory for patch files, organized by namespace/spec_id

CONFIG patches.auto_cleanup:
  type: bool
  default: false
  env: NLSPEC_PATCHES_AUTO_CLEANUP
  description: If true, delete patch files after ABSORBED status
```

### CONFIG: validation (Phase 2)

```
CONFIG validation.strict:
  type: bool
  default: false
  env: NLSPEC_VALIDATION_STRICT
  description: When true, warnings are treated as errors in nlspec_validate

CONFIG validation.max_undeclared_sections:
  type: u16
  default: 3
  env: NLSPEC_MAX_UNDECLARED_SECTIONS
  description: Maximum undeclared sections before flagging as gap
```

### CONFIG: slice (Phase 2)

```
CONFIG slice.max_depth:
  type: u64
  default: 3
  env: NLSPEC_SLICE_MAX_DEPTH
  description: Maximum reference traversal depth for context slicing

CONFIG slice.max_elements:
  type: u64
  default: 500
  env: NLSPEC_SLICE_MAX_ELEMENTS
  description: Maximum elements per slice to prevent runaway traversals
```

### CONFIG: storage (Phase 2)

```
CONFIG storage.backend:
  type: String
  default: "local"
  env: NLSPEC_STORAGE_BACKEND
  description: Storage backend for spec files. "local" reads from filesystem, "s3" reads from S3 bucket.
  values: "local" | "s3"

CONFIG storage.watch:
  type: bool
  default: true
  env: NLSPEC_STORAGE_WATCH
  description: Enable file watching for cache invalidation

CONFIG storage.s3.bucket:
  type: String | None
  default: None
  env: NLSPEC_S3_BUCKET
  description: S3 bucket name when storage backend is "s3"

CONFIG storage.s3.prefix:
  type: String
  default: "specs/"
  env: NLSPEC_S3_PREFIX
  description: S3 key prefix for spec files

CONFIG storage.s3.region:
  type: String
  default: "us-east-1"
  env: NLSPEC_S3_REGION
  description: AWS region for S3 bucket

CONFIG storage.s3.credentials:
  type: String | None
  default: None
  env: AWS_PROFILE
  description: AWS profile name for S3 access
```

### CONFIG: substrate (Phase 2)

```
CONFIG substrate.cache_ttl_ms:
  type: u64
  default: 300000
  env: NLSPEC_SUBSTRATE_CACHE_TTL
  description: Time-to-live for in-memory substrate graph cache (milliseconds)

CONFIG substrate.branching_detection:
  type: String
  default: "derives_from"
  env: NLSPEC_SUBSTRATE_BRANCHING
  description: How to detect substrate branching. "derives_from" or "layer"
  values: "derives_from" | "layer"
```

### CONFIG: import (Phase 2)

```
CONFIG import.auto_namespace:
  type: bool
  default: true
  env: NLSPEC_IMPORT_AUTO_NAMESPACE
  description: When true, nlspec_import creates namespaces automatically

CONFIG import.follow_imports:
  type: bool
  default: false
  env: NLSPEC_IMPORT_FOLLOW_IMPORTS
  description: When true, nlspec_import recursively imports all specs this spec imports from

CONFIG import.verify_layer_derivation:
  type: bool
  default: true
  env: NLSPEC_IMPORT_VERIFY_LAYER
  description: When true, import validates DERIVES FROM parent is already imported
```

### CONFIG: git (Phase 2)

```
CONFIG git.enabled:
  type: bool
  default: true
  env: NLSPEC_GIT_ENABLED
  description: Enable git integration (version tracking, blame, etc.)

CONFIG git.repo_path:
  type: String | None
  default: None
  env: NLSPEC_GIT_REPO_PATH
  description: Git repository path. If not set, auto-detects from project_dir.

CONFIG git.auto_commit:
  type: bool
  default: false
  env: NLSPEC_GIT_AUTO_COMMIT
  description: If true, automatically commit changes to markdown files
```

---

## Deployment Artifacts

### Distribution (Phase 1)

```
PyPI:   pip install nlspec-server              (local dev, stdio transport, SQLite)
GitHub: releases with pre-built binaries for macOS, Linux, Windows
```

### Distribution (Phase 2)

```
PyPI:   pip install nlspec-server              (MCP + gRPC)
Docker: ghcr.io/nlspec/nlspec-server:latest    (production, SSE + gRPC)
GitHub: releases with pre-built binaries
```

### Artifact: Python Package

```
Distribution: PyPI via uv/pip
Package name: nlspec-server
Entry points:
  nlspec-server (CLI)
  nlspec.server (Python module)

Core Dependencies (Phase 1):
  - fastmcp (MCP server framework)
  - sqlalchemy (ORM, supports SQLite + PostgreSQL)
  - pydantic (validation)
  - python-dotenv (env var loading)
  - watchdog (file watcher)

Phase 2 Additional Dependencies:
  - psycopg2-binary (PostgreSQL driver, required for Phase 2)
  - grpcio (gRPC server)
  - protobuf (protobuf serialization)
  - boto3 (S3 support, optional)

Optional Dependencies:
  - psycopg2-binary (PostgreSQL driver, optional for Phase 1)
```

### Artifact: Docker Image (Phase 2)

```
Registry: ghcr.io/nlspec/nlspec-server
Tags:
  - :latest (main branch)
  - :v2.0.0 (release tags)
Entrypoint: nlspec-server
Environment:
  - All config via env vars
  - Volume mounts: /specs (specs), /patches (patches)
Build:
  FROM python:3.11-slim
  RUN apt-get update && apt-get install -y postgresql-client git
  COPY . /app
  RUN uv pip install /app
  EXPOSE 8080 50060
  ENTRYPOINT ["nlspec-server"]
```

### Artifact: Docker Compose (Phase 2)

```yaml
version: '3.9'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: nlspec
      POSTGRES_USER: nlspec
      POSTGRES_PASSWORD: nlspec
    volumes:
      - db-data:/var/lib/postgresql/data

  server:
    image: ghcr.io/nlspec/nlspec-server:latest
    ports:
      - "8080:8080"
      - "50060:50060"
    environment:
      DATABASE_URL: postgresql://nlspec:nlspec@db:5432/nlspec
      NLSPEC_TRANSPORT: sse
      NLSPEC_HOST: 0.0.0.0
      NLSPEC_GRPC_PORT: 50060
    volumes:
      - ./specs:/specs
      - ./patches:/patches
    depends_on:
      - db

volumes:
  db-data:
```

### Deployment Strategy

**Phase 1 is local-only:** The MCP server runs as a subprocess on the developer's machine. No Docker, no cloud deployment. Configuration is environment variables + `.env` file.

**Phase 2 is cloud-ready:** The server can run via Docker, with PostgreSQL, SSE transport, and gRPC for internal services.

---

## FileStructure

```
nlspec/
├── pyproject.toml
├── README.md
├── CLAUDE.md
├── .env.example
├── docker-compose.yml (Phase 2)
├── Dockerfile (Phase 2)
├── specs/
│   ├── L1/
│   │   └── L1-nlspec-server-spec.md
│   ├── L2/
│   │   └── L2-nlspec-workflows-spec.md
│   ├── L3/
│   │   └── L3-nlspec-config.md
│   └── L4/
│       └── L4-nlspec-seed-spec.md (Phase 2)
├── patches/
│   └── nlspec/                           -- patches for nlspec's own specs
│       ├── l1-bootstrap/
│       ├── l2-workflows/
│       ├── l3-config/
│       └── l4-seed/
├── .nlspec/
│   ├── index.sqlite                -- SQLite fallback index (auto-created)
│   ├── config.yaml                 -- optional config file
│   └── namespaces.json (Phase 2)   -- namespace registry
├── src/
│   └── nlspec_server/
│       ├── __init__.py
│       ├── server.py               -- FastMCP server entry point
│       ├── cli.py                  -- CLI entry point
│       ├── config.py               -- configuration loading
│       ├── models/
│       │   ├── __init__.py
│       │   ├── types.py            -- all records
│       │   └── errors.py           -- error hierarchy
│       ├── parser/
│       │   ├── __init__.py
│       │   ├── spec_parser.py
│       │   ├── element_patterns.py
│       │   └── renderer.py
│       ├── store/
│       │   ├── __init__.py
│       │   ├── spec_store.py
│       │   ├── sqlite_index.py
│       │   ├── postgres_index.py
│       │   └── markdown_writer.py
│       ├── query/
│       │   ├── __init__.py
│       │   ├── query_engine.py
│       │   ├── fts_setup.py
│       │   └── ranking.py
│       ├── namespace/ (Phase 2)
│       │   ├── __init__.py
│       │   └── namespace_manager.py
│       ├── slicer/ (Phase 2)
│       │   ├── __init__.py
│       │   └── context_slicer.py
│       ├── patch/ (Phase 2)
│       │   ├── __init__.py
│       │   ├── patch_manager.py
│       │   └── patch_loader.py
│       ├── validator/ (Phase 2)
│       │   ├── __init__.py
│       │   ├── spec_validator.py
│       │   └── gap_analyzer.py
│       ├── graph/ (Phase 2)
│       │   ├── __init__.py
│       │   ├── graph_engine.py
│       │   └── substrate_resolver.py
│       ├── splitter/ (Phase 2)
│       │   ├── __init__.py
│       │   ├── spec_splitter.py
│       │   └── clustering.py
│       ├── drift/ (Phase 2)
│       │   ├── __init__.py
│       │   └── drift_detector.py
│       ├── mcp/
│       │   ├── __init__.py
│       │   ├── server.py
│       │   └── tools/
│       │       ├── __init__.py
│       │       ├── init.py
│       │       ├── get.py
│       │       ├── list.py
│       │       ├── search.py
│       │       ├── create.py
│       │       ├── update.py
│       │       ├── delete.py
│       │       ├── query.py
│       │       ├── import.py (Phase 2)
│       │       ├── slice.py (Phase 2)
│       │       ├── patch.py (Phase 2)
│       │       ├── validate.py (Phase 2)
│       │       ├── graph.py (Phase 2)
│       │       ├── split.py (Phase 2)
│       │       ├── drift.py (Phase 2)
│       │       ├── layer.py (Phase 2)
│       │       └── substrate.py (Phase 2)
│       ├── grpc/ (Phase 2)
│       │   ├── __init__.py
│       │   ├── server.py
│       │   └── services.py
│       └── storage/
│           ├── __init__.py
│           ├── local_storage.py
│           ├── s3_storage.py (Phase 2, optional)
│           └── file_watcher.py
├── tests/
│   ├── fixtures/
│   │   ├── kv-store-spec.md
│   │   └── auth-spec.md
│   ├── scenarios/
│   │   ├── test_scenario_01_init.py
│   │   ├── test_scenario_02_parse.py
│   │   ├── test_scenario_03_search.py
│   │   ├── test_scenario_04_create.py
│   │   ├── test_scenario_05_update.py
│   │   ├── test_scenario_06_list.py
│   │   ├── test_scenario_07_search_text.py
│   │   ├── test_scenario_08_import.py (Phase 2)
│   │   ├── test_scenario_09_validate.py (Phase 2)
│   │   ├── test_scenario_10_slice.py (Phase 2)
│   │   ├── test_scenario_12_patch.py (Phase 2)
│   │   ├── test_scenario_15_drift.py (Phase 2)
│   │   └── test_scenario_19_substrate.py (Phase 2)
│   └── smoke/
│       └── test_smoke.py
├── templates/
│   └── nlspec-template.md
└── docs/
    ├── API.md
    ├── CONFIGURATION.md (this layer)
    ├── DEPLOYMENT.md (Phase 2)
    └── CONTRIBUTING.md
```

---

## Build and Run

### Prerequisites (Phase 1)

- Python 3.11+
- SQLite (built-in, no setup required)
- PostgreSQL 13+ (optional; install only if you want PostgreSQL instead of SQLite)
- uv (recommended) or pip

### Prerequisites (Phase 2)

- Python 3.11+
- PostgreSQL 13+ (required for Phase 2)
- uv (recommended) or pip
- Docker + Docker Compose (optional; for container deployment)

### Install (Local Dev)

```bash
# Clone repository
git clone https://github.com/divyendudeepakingh/nlspec.git
cd nlspec

# Install in development mode
uv pip install -e ".[dev]"

# Copy env template
cp .env.example .env
```

### Run Tests

```bash
# All tests
pytest

# Smoke tests only
pytest tests/smoke/ -v

# Specific scenario
pytest tests/scenarios/test_scenario_01_init.py -v

# With coverage
pytest --cov=nlspec_server tests/
```

### Run as MCP Server (Phase 1, stdio)

**Simplest (uses SQLite):**
```bash
nlspec-server --specs-dir ./specs
```

**With explicit config:**
```bash
export NLSPEC_PROJECT_DIR="."
export NLSPEC_SPECS_DIR="./specs"
export NLSPEC_DATABASE_TYPE="sqlite"
export NLSPEC_SQLITE_PATH=".nlspec/index.sqlite"
export NLSPEC_FTS=true
nlspec-server
```

**With PostgreSQL (optional):**
```bash
export DATABASE_URL="postgresql://user:pass@localhost:5432/nlspec"
export NLSPEC_DATABASE_TYPE="postgresql"
nlspec-server --specs-dir ./specs
```

### Run as MCP Server (Phase 2, with PostgreSQL + gRPC)

```bash
export DATABASE_URL="postgresql://user:pass@localhost:5432/nlspec"
export NLSPEC_TRANSPORT="sse"
export NLSPEC_GRPC_PORT="50060"
nlspec-server --specs-dir ./specs
```

### Run in Docker (Phase 2)

```bash
docker-compose up -d
```

Server is available at:
- MCP: stdio (via Claude Code configured with `.mcp.json`)
- SSE: http://localhost:8080
- gRPC: localhost:50060

### Verify MCP Connection (Local Dev)

1. Add to `.mcp.json` in your project:
   ```json
   {
     "mcpServers": {
       "nlspec": {
         "command": "nlspec-server",
         "args": ["--specs-dir", "./specs"]
       }
     }
   }
   ```

2. In Claude Code: "List all specs"
   - Agent calls `nlspec_list({})`
   - Should return specs in index

---

## Dependencies

### Runtime Dependencies (Phase 1)

```
fastmcp           >= 1.0.0    -- MCP server framework
sqlalchemy        >= 2.0.0    -- database ORM (SQLite default, PostgreSQL optional)
pydantic          >= 2.0.0    -- data validation
python-dotenv     >= 1.0.0    -- env var loading
watchdog          >= 3.0.0    -- file watcher
```

### Runtime Dependencies (Phase 2)

```
fastmcp           >= 1.0.0    -- MCP server framework
sqlalchemy        >= 2.0.0    -- database ORM (PostgreSQL + SQLite)
psycopg2-binary   >= 2.9.0    -- PostgreSQL driver (required for Phase 2)
pydantic          >= 2.0.0    -- data validation
python-dotenv     >= 1.0.0    -- env var loading
watchdog          >= 3.0.0    -- file watcher
grpcio            >= 1.50.0   -- gRPC server
protobuf          >= 4.0.0    -- protobuf serialization
boto3             >= 1.26.0   -- AWS S3 support (optional)
```

### Optional Runtime Dependencies

```
psycopg2-binary   >= 2.9.0    -- PostgreSQL driver (only if using PostgreSQL, not required for Phase 1)
boto3             >= 1.26.0   -- S3 storage backend (Phase 2+ only)
```

### Development Dependencies

```
pytest            >= 7.0.0    -- test runner
pytest-cov        >= 4.0.0    -- coverage
pytest-asyncio    >= 0.21.0   -- async test support
black             >= 23.0.0    -- code formatter
flake8            >= 6.0.0    -- linter
mypy              >= 1.0.0    -- type checker
```

---

## Appendix: Environment Variables Reference (Phase 1)

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| NLSPEC_TRANSPORT | stdio | String | MCP transport (Phase 1: "stdio") |
| NLSPEC_PROJECT_DIR | . | String | project root directory |
| NLSPEC_SPECS_DIR | . | String | specs directory path |
| NLSPEC_DATABASE_TYPE | sqlite | String | Database backend (sqlite, postgresql) |
| DATABASE_URL | None | String | PostgreSQL connection URL (optional) |
| NLSPEC_SQLITE_PATH | .nlspec/index.sqlite | String | SQLite path |
| NLSPEC_AUTO_REINDEX | true | bool | Re-parse on file changes |
| NLSPEC_FTS | true | bool | Enable full-text search |
| NLSPEC_MIN_RELEVANCE | 0.1 | float | Minimum relevance score for search |

---

## Appendix: Environment Variables Reference (Phase 2)

| Variable | Default | Type | Description |
|----------|---------|------|-------------|
| NLSPEC_TRANSPORT | stdio | String | MCP transport ("stdio", "sse") |
| NLSPEC_HOST | 127.0.0.1 | String | Host for SSE (if transport="sse") |
| NLSPEC_PORT | 8080 | u16 | Port for SSE (if transport="sse") |
| NLSPEC_GRPC_PORT | 50060 | u16 | gRPC port |
| NLSPEC_DATABASE_TYPE | postgresql | String | Database backend (postgresql, sqlite) |
| DATABASE_URL | required | String | PostgreSQL connection URL |
| NLSPEC_DEFAULT_NAMESPACE | default | String | Default namespace |
| NLSPEC_AUTO_CREATE_NAMESPACE | true | bool | Auto-create namespaces |
| NLSPEC_PATCHES_DIR | patches/ | String | Directory for patch files |
| NLSPEC_SLICE_MAX_DEPTH | 3 | u64 | Max reference traversal depth |
| NLSPEC_SLICE_MAX_ELEMENTS | 500 | u64 | Max elements per slice |
| NLSPEC_SUBSTRATE_CACHE_TTL | 300000 | u64 | Substrate graph cache TTL (ms) |
| NLSPEC_GIT_ENABLED | true | bool | Enable git integration |
| NLSPEC_GIT_AUTO_COMMIT | false | bool | Auto-commit changes |
| NLSPEC_STORAGE_BACKEND | local | String | Storage backend (local, s3) |
| NLSPEC_S3_BUCKET | None | String | S3 bucket name (if backend="s3") |

---

## Appendix: Revision History

```
| Version | Date       | Author                  | Changes                              |
|---------|------------|-------------------------|--------------------------------------|
| 1.0.0   | Mar 2026   | Divyendu Deepak Singh   | Merged Phase 1 + Phase 2 Config     |
```
