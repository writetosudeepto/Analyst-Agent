# Analyst Agent with MCP Server - Research & Architecture Plan

## Project Goal

Build an open-source Analyst Agent that can query databases using natural language (Text-to-SQL), given data schema, relationships, metadata, and business metrics — powered by MCP (Model Context Protocol) server support.

---

## 1. Landscape Research: Open-Source Analyst Agent Frameworks

### 1.1 Frameworks Evaluated

| Framework | Type | License | Stars | Key Strength |
|-----------|------|---------|-------|--------------|
| **Vanna AI 2.0** | Text-to-SQL Agent | MIT | 20k+ | RAG-powered SQL gen, agent-based architecture, multi-tenant |
| **Wren AI** | GenBI + Semantic Engine | AGPL-3.0 | 10k+ | Semantic MDL modeling, privacy-first (no data sent to LLM) |
| **dbt + MetricFlow** | Semantic Layer + MCP | Apache 2.0 | 10k+ | Governed metrics, MCP server, lineage, manifest artifacts |
| **Cube.dev** | Semantic Layer + Agentic BI | MIT (core) | 18k+ | Semantic SQL proxy, MCP + A2A support, governance guardrails |
| **LangChain SQL Agent** | Agent Toolkit | MIT | 100k+ | SQLDatabaseToolkit, ReAct agent, LangGraph integration |
| **MAC-SQL** | Multi-Agent Research | Apache 2.0 | Research | Selector→Decomposer→Refiner pipeline for complex queries |
| **SQLMesh** | Data Transformation | Apache 2.0 | 2k+ | AST-level SQL understanding, column-level lineage |
| **Google MCP Toolbox** | MCP Database Server | Apache 2.0 | New | Multi-DB, connection pooling, auth, plain English queries |
| **DBHub** | Universal DB MCP Server | MIT | 2k+ | Any MCP client, multi-DB, read-only safety mode |

### 1.2 Key MCP Database Servers (Open Source)

| Project | Databases | Language | Key Feature |
|---------|-----------|----------|-------------|
| `googleapis/genai-toolbox` | Postgres, MySQL, SQLite, AlloyDB | Go | Google-backed, connection pooling, auth |
| `FreePeak/db-mcp-server` | MySQL, Postgres, SQLite | Go | Multi-DB simultaneous, OpenAI Agents SDK compat |
| `benborla/mcp-server-mysql` | MySQL | TypeScript | SSH tunnel support, Claude Code native |
| `executeautomation/mcp-database-server` | SQLite, MSSQL, Postgres, MySQL | TypeScript | Multi-DB via npx |
| `RichardHan/mssql_mcp_server` | MSSQL | Python | SQL/Windows/Azure auth |
| `mcp-server-datahub` | Any (via DataHub) | Python | Business glossaries, metrics, lineage, domains |
| `database-mcp` (PyPI) | Postgres, MySQL, MSSQL | Python | Zero-config schema discovery |
| `dbt MCP Server` | Any (via dbt) | Python | Governed metrics, semantic layer, CLI tools |

---

## 2. Architecture Patterns for Text-to-SQL Agents

### 2.1 Pattern A: Direct Text-to-SQL (LangChain Style)

```
User Question → LLM → Raw SQL → Database → Results → LLM → Answer
```

- **Pros:** Simple, fast to prototype
- **Cons:** No business context, hallucination-prone, inconsistent metric calculations
- **Best for:** Internal tools, dev/debug, simple schemas

### 2.2 Pattern B: RAG-Enhanced Text-to-SQL (Vanna Style)

```
User Question → Embedding Search (schema + docs + past queries)
             → LLM + Retrieved Context → SQL → Database → Results → Answer
```

- **Pros:** Learns from past queries, adapts to domain language
- **Cons:** Needs training data, no formal metric definitions
- **Best for:** Teams with query history, medium complexity schemas

### 2.3 Pattern C: Semantic Layer Proxy (Cube/dbt/Wren Style)

```
User Question → LLM → Semantic SQL (against governed metrics/models)
             → Semantic Layer Runtime → Optimized Raw SQL → Database → Results
```

- **Pros:** Governed metrics, consistent calculations, security enforcement
- **Cons:** Requires upfront semantic modeling effort
- **Best for:** Enterprise, multi-team, mission-critical analytics

### 2.4 Pattern D: Multi-Agent Collaborative (MAC-SQL Style)

```
User Question → Selector Agent (prune schema)
             → Decomposer Agent (break into sub-questions)
             → SQL Generator → Refiner Agent (execute + fix errors)
             → Final SQL → Database → Results
```

- **Pros:** Handles complex queries, error self-correction, works with smaller LLMs
- **Cons:** Higher latency, more complex orchestration
- **Best for:** Complex schemas (100+ tables), analytical queries with joins/subqueries

---

## 3. Recommended Architecture for This Project

### Hybrid Approach: Semantic-Aware Multi-Agent with MCP

Combines the best of all patterns:

```
┌─────────────────────────────────────────────────────────────────┐
│                        MCP Server Layer                         │
│  (Exposes tools via Model Context Protocol)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │ Schema Tool   │  │ Metrics Tool │  │ Query Executor Tool│    │
│  │ - list_tables │  │ - list_metrics│  │ - execute_sql      │    │
│  │ - get_schema  │  │ - get_metric │  │ - validate_sql     │    │
│  │ - get_relations│ │ - calc_rules │  │ - explain_query    │    │
│  │ - get_samples │  │ - dimensions │  │ - get_results      │    │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────┘    │
│         │                  │                     │               │
│  ┌──────┴──────────────────┴─────────────────────┴──────────┐   │
│  │              Semantic Context Engine                       │   │
│  │  - Schema registry (tables, columns, types, relations)    │   │
│  │  - Business glossary (metric definitions, calculations)   │   │
│  │  - Metadata store (descriptions, tags, owners)            │   │
│  │  - Query history + RAG index (learned patterns)           │   │
│  └──────────────────────────┬────────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────────────────┴────────────────────────────────┐   │
│  │              Database Connector Layer                      │   │
│  │  - SQLAlchemy (Postgres, MySQL, SQLite, etc.)             │   │
│  │  - Connection pooling, read-only mode, timeout controls   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      Agent Orchestrator                          │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐     │
│  │ Schema       │  │ SQL Generator│  │ SQL Refiner         │     │
│  │ Selector     │  │ Agent        │  │ Agent               │     │
│  │              │  │              │  │                     │     │
│  │ Picks relevant│ │ Generates SQL│  │ Executes, catches   │     │
│  │ tables/cols  │  │ using context│  │ errors, rewrites    │     │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬───────────┘     │
│         │                  │                     │                │
│  ┌──────┴──────────────────┴─────────────────────┴────────────┐  │
│  │                   LLM Provider                              │  │
│  │  Claude 4.5/4.6 | GPT-5 | Llama | Any OpenAI-compat       │  │
│  └─────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Implementation Plan

### Phase 1: Core MCP Server + Schema Engine
- [ ] Project scaffolding (Python, pyproject.toml, uv/poetry)
- [ ] Database connector layer (SQLAlchemy, multi-DB support)
- [ ] Schema introspection engine (tables, columns, types, foreign keys, indexes)
- [ ] MCP server with `mcp` Python SDK (stdio + SSE transport)
- [ ] Core MCP tools: `list_tables`, `get_table_schema`, `get_relationships`, `get_sample_data`

### Phase 2: Semantic Context Layer
- [ ] YAML-based semantic model definition (metrics, dimensions, relationships, descriptions)
- [ ] Business glossary store (metric definitions, calculation rules, aliases)
- [ ] Metadata enrichment (column descriptions, business terms, data types)
- [ ] MCP tools: `list_metrics`, `get_metric_definition`, `search_glossary`

### Phase 3: Text-to-SQL Agent
- [ ] Schema selector agent (prune irrelevant tables/columns for the question)
- [ ] SQL generator with semantic context injection
- [ ] SQL validator and refiner (execute, catch errors, rewrite)
- [ ] Query result formatter (summaries for LLM, rich output for user)
- [ ] MCP tools: `generate_sql`, `execute_query`, `explain_results`

### Phase 4: RAG & Learning Loop
- [ ] Vector store for schema embeddings + past queries (ChromaDB/FAISS)
- [ ] Query history tracking and similarity search
- [ ] Few-shot example retrieval for improved SQL generation
- [ ] Feedback loop: user corrections improve future generations

### Phase 5: Production Hardening
- [ ] Read-only mode and query allow/deny lists
- [ ] Row-level security and user context propagation
- [ ] Connection pooling and query timeout controls
- [ ] Streaming responses (SSE)
- [ ] Docker deployment + docker-compose with sample database

---

## 5. Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Python 3.11+ | Ecosystem maturity, LLM library support |
| MCP SDK | `mcp` (official Python SDK v1.x) | Anthropic's official protocol, stable |
| Database | SQLAlchemy 2.0 | Multi-DB support, async, mature ORM |
| LLM | Anthropic Claude SDK / LiteLLM | Claude 4.5/4.6 primary, LiteLLM for flexibility |
| Vector Store | ChromaDB | Lightweight, embedded, good for RAG |
| Semantic Models | YAML (custom MDL) | Declarative, version-controllable, dbt-compatible |
| Testing | pytest + pytest-asyncio | Standard Python testing |
| Packaging | pyproject.toml + uv | Modern Python packaging |

---

## 6. Key Design Decisions

### Why MCP over direct API integration?
- **Standardized protocol** — works with Claude Desktop, Cursor, VS Code, any MCP client
- **Tool discovery** — clients auto-discover available capabilities
- **Security** — controlled interface, no direct DB credentials exposed to LLM
- **Composability** — can be combined with other MCP servers (file, web, etc.)

### Why a Semantic Layer matters for Text-to-SQL
- Without it, LLMs guess at metric calculations (e.g., "revenue" = `SUM(amount)` vs `SUM(amount * quantity)`)
- Business terms are ambiguous — "active user" means different things in different contexts
- The semantic layer provides deterministic, governed definitions that eliminate hallucination
- Schema descriptions and relationships reduce token waste on LLM reasoning

### Why Multi-Agent over Single-Agent?
- Complex queries with 100+ table schemas overflow context windows
- Schema selection reduces noise and improves accuracy
- Error self-correction via execution feedback catches 80%+ of SQL errors
- Specialized agents can use different prompting strategies

---

## 7. Reference Projects to Study

| Project | URL | What to Learn |
|---------|-----|---------------|
| Vanna AI | https://github.com/vanna-ai/vanna | RAG architecture, training pipeline, multi-LLM support |
| Wren AI | https://github.com/Canner/WrenAI | Semantic MDL modeling, engine architecture |
| dbt MCP Server | https://github.com/dbt-labs/dbt-mcp | MCP tool patterns, semantic layer integration |
| MCP Python SDK | https://github.com/modelcontextprotocol/python-sdk | MCP server implementation patterns |
| Google MCP Toolbox | https://github.com/googleapis/genai-toolbox | Multi-DB MCP server, connection pooling |
| DBHub | https://github.com/bytebase/dbhub | Universal DB MCP, read-only safety |
| DataHub MCP | https://pypi.org/project/mcp-server-datahub/ | Business glossary, metrics via MCP |
| MAC-SQL | https://github.com/wbbeyourself/MAC-SQL | Multi-agent text-to-SQL pipeline |
| LangChain SQL Agent | https://docs.langchain.com/oss/python/langgraph/sql-agent | LangGraph SQL agent patterns |
| Cube.dev | https://github.com/cube-js/cube | Semantic SQL proxy, governance |

---

## 8. Success Criteria

1. **Accuracy**: Generated SQL returns correct results for 80%+ of natural language queries on test schemas
2. **Schema Awareness**: Agent correctly identifies relevant tables/columns without hallucinating non-existent ones
3. **Metric Consistency**: Same business question always produces the same calculation logic
4. **Error Recovery**: Agent self-corrects SQL errors via execution feedback in 90%+ of recoverable cases
5. **MCP Compliance**: Server works with Claude Desktop, Cursor, and any MCP-compatible client
6. **Security**: No raw data sent to LLM, read-only mode default, query validation
7. **Extensibility**: New databases, metrics, and tools can be added via config/YAML without code changes
