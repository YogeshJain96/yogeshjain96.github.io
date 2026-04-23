---
title: "Building real-time, data-aware intelligence with Postgres and the Model Context Protocol"
date: 2026-04-23T06:51:00-07:00
description: "Eliminating Hallucinations in LLMs with Postgres and the Model Context Protocol"
tags: ["postgres", "MCP", "tech", "model-context-protocol"]
cover:
  image: images/postgres-and-mcp/cover.png
---

Ask an LLM to write SQL against your production database. It'll generate something syntactically perfect. It'll also reference tables that don't exist, columns that were renamed six months ago, and JOIN conditions it made up from training data. The query fails. Or worse, it runs and returns the wrong answer.

I tried to resolve this, and the path led me to the Model Context Protocol (MCP) and a Postgres-native approach to eliminating SQL hallucinations.

## Where it all started

In early 2024, before MCP or AI agents were a thing, I wanted something simple: a chatbot that could answer questions using live data from APIs (or SQL behind the API). So I hacked something together.

1. Passed the OpenAPI spec to an LLM.
2. Asked it to generate ONLY a `curl` command as response.
3. Executed it using a GO function.
4. Feed those response back to the LLM for a human-readable answer.

```sh
User: "What's the CPU usage and total DB connections right now?"
  -> LLM + API Spec -> curl -X GET https://portal.curiousone.in/api/v1/metrics?names=cpu,db_conn
  -> execute() -> { cpu: 72%, conn: 84 }
  -> LLM: "CPU is at 72% and there are 84 active DB connections."
```

It worked. Natural language in, real data out.

But it didn't scale. Every new data source needed a new parser. The prompt engineering was fragile, specific to each spec/ schema.

When I spoke about my idea to people I met during many conferences, they were building something similar for this kind of use case and facing the same challenge.

Then Anthropic released MCP as an open standard, and it solved exactly this problem.

## The Problem - LLMs are blind to your data

So LLMs understand SQL syntax. They can write complex queries with CTEs, subqueries. What they don't know is _your_ table names, column names, or relationships. They guess based on patterns from training data.

In practice:

| What the LLM generates                                                  | What actually exists                                                                                    |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `SELECT u.name, o.total FROM users u JOIN orders o ON u.id = o.user_id` | `SELECT c.full_name, o.total_price FROM app_customers c JOIN customer_orders o ON c.id = o.customer_id` |
| **ERROR:** relation "users" does not exist                              | **SUCCESS:** 47 rows returned                                                                           |

The LLM guessed `users` when the real table is `app_customers`. Guessed `name` when the column is `full_name`. Guessed `o.user_id` when the foreign key is `o.customer_id`. Every guess was reasonable. Every guess was wrong.

### The context gap

Every SQL hallucination traces back to missing context:

| Hallucination | Example                     | Missing context                        |
| ------------- | --------------------------- | -------------------------------------- |
| Wrong table   | `FROM users`                | Real table is `app_customers`          |
| Wrong column  | `SELECT email FROM orders`  | Column doesn't exist on that table     |
| Wrong JOIN    | `ON users.id = orders.id`   | Should be `orders.user_id`             |
| Wrong types   | `WHERE created_at = '2024'` | It's a `timestamptz`, not a string     |
| Wrong dialect | `DATEADD(day, 7, now())`    | That's SQL Server syntax, not Postgres |

The fix isn't better prompting. It's giving the LLM access to the **actual database schema** at query time.

## The Solution - The Model Context Protocol

MCP is an open standard that connects AI models to external data sources and tools. One protocol, any data source - **think USB-C for AI**.

The **key architectural insight**: a single MCP client connects to _multiple_ MCP servers, each wrapping a different data source. For Postgres operations, this means one AI assistant can simultaneously access multiple data source, for example:

```sh
                    ┌─ Postgres MCP Server ──→ PostgreSQL (live schema & queries)
                    │
MCP Client ─────────┼─ Filesystem MCP Server ─→ PG Logs (log file analysis)
(Claude, IDEs)      │
                    └─ Prometheus MCP Server ─→ Telemetry (metrics & alerting data)
```

The client is the AI application. Each server exposes tools the LLM can call. They all talk over JSON-RPC 2.0 via stdio or SSE transport. The LLM decides which server to call based on the question - a schema question goes to Postgres MCP, a **"why was the database slow last night"** question might hit both the Filesystem MCP (for logs) and Prometheus MCP (for metrics).

### Building blocks

MCP has three primitives. For databases, only one really matters.

- **Tools** are functions the LLM can call that return live results. `execute_sql`, `list_schemas`, `list_objects` -- these return current data, not stale snapshots.

- **Resources** are read-only static data served via URIs. Schema snapshots published as resources go stale fast, making them a poor fit for database metadata.

- **Prompts** are reusable templates that bundle context and instructions. Useful, but secondary for database work.

## Postgres + MCP: pg-airman-mcp

[pg-airman-mcp](https://github.com/EnterpriseDB/pg-airman-mcp) is a Postgres MCP server that gives LLMs live access to your database's structure and data. Here are some of the tools:

| Tool                    | What it does                                                                            |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `list_schemas`          | All database schemas, categorized as system or user. Starting point for discovery.      |
| `list_objects`          | Tables, views, sequences, and functions within a schema, including comments.            |
| `get_object_details`    | Columns, types, constraints, indexes, and comments for any object.                      |
| `explain_query`         | Runs EXPLAIN plans and simulates hypothetical indexes via HypoPG without creating them. |
| `execute_sql`           | Runs queries with configurable access control, read-only mode, and safe SQL parsing.    |
| `analyze_query_indexes` | Explores thousands of possible indexes to find optimal solutions for a workload.        |

### The query flow

When a user asks **`"Show me top 5 customers by revenue"`** the LLM doesn't guess. It discovers step by step:

1. Calls `list_schemas`: discovers available schemas
2. Calls `list_objects`: gets real table and view names
3. Calls `get_object_details`: learns columns, types, constraints, indexes
4. Calls `explain_query`: validates the query plan, checks performance
5. Calls `execute_sql`: runs the query with access control and safety parsing

By step 3, the LLM knows the table is `app_customers`, the column is `full_name`, and the foreign key is `customer_id`. **No guessing.**

### pg_catalog: Postgres knows itself

Here's what makes this work so well. Postgres already has everything you need in its own metadata catalogs. The MCP server doesn't need external config files, schema dumps, or documentation you forgot to update. It just asks Postgres.

- `pg_class`: all tables, views, sequences
- `pg_attribute`: column names and types
- `pg_constraint`: PKs, FKs, CHECK constraints
- `pg_index`: indexes for query optimization
- `pg_namespace`: schemas
- `pg_description`: `COMMENT ON` annotations (which double as documentation)

Under the hood, the MCP server runs queries like these:

```sql
-- list_schemas
SELECT
  schema_name, schema_owner,
  CASE
    WHEN schema_name LIKE 'pg_%' THEN 'System Schema'
    WHEN schema_name = 'information_schema' THEN 'System Info Schema'
    ELSE 'User Schema'
  END AS schema_type
FROM information_schema.schemata
ORDER BY schema_type, schema_name;
```

```sql
-- list_objects
SELECT
  CASE c.relkind
    WHEN 'r' THEN 'table'
    WHEN 'v' THEN 'view'
  END AS object_type,
  n.nspname AS table_schema,
  c.relname AS table_name,
  d.description AS comment
FROM pg_catalog.pg_class c
JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_catalog.pg_description d
  ON d.objoid = c.oid AND d.objsubid = 0
WHERE n.nspname = $1 AND c.relkind IN ('r','v')
ORDER BY c.relname;
```

```sql
-- get_object_details
SELECT
  column_name, data_type,
  is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = $1
  AND table_name = $2
ORDER BY ordinal_position;
```

These run against the live database. Column added, renamed, dropped? The MCP server reflects it immediately in real time.

### Optimizing queries with EXPLAIN

Getting the right tables and columns is the initial step. Writing a fast query is the next step.

The `explain_query` tool feeds the LLM's generated SQL through Postgres query planner. If EXPLAIN shows a unoptimized plan, the LLM can rewrite the query before it ever executes in few ms, without the user even realizing.

**For example:** The LLM's initial attempt

```sql
SELECT c.full_name, SUM(o.total_price)
FROM   app_customers c
JOIN   customer_orders o ON c.id = o.customer_id
GROUP BY c.full_name
ORDER BY total DESC;
```

**EXPLAIN reveals:** sequential scan on **~500K rows**, no date filter (scanning every order ever placed), **~2,999 ms** execution time.

The LLM reads the EXPLAIN output and rewrites the SQL:

```sql
SELECT c.full_name, SUM(o.total_price)
FROM   app_customers c
JOIN   customer_orders o ON c.id = o.customer_id
WHERE  o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY c.id, c.full_name
ORDER BY total DESC
LIMIT 10;
```

Index scan on **~3,000 rows. 20 ms.** That's **~150x faster.** The LLM **added a date filter**, fixed the GROUP BY (included `c.id` for correctness), and **added a LIMIT**. All from reading the EXPLAIN plan, not from memorized optimization rules.

### Guard rails

Giving an LLM database access requires security at multiple layers. Here's one of the recommend setup.

Start with a **read-only role**. `GRANT SELECT` only. If the LLM generates a `DROP TABLE`, Postgres rejects it at the engine level.

```sql
CREATE ROLE mcp_reader LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE mydb TO mcp_reader;
GRANT USAGE ON SCHEMA public TO mcp_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_reader;
```

For multi-tenant databases, use Row-Level Security (RLS). It enforces isolation at the Postgres engine level. SQL injection can't bypass it.

```sql
ALTER TABLE customer_orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON customer_orders
  FOR SELECT USING (
    tenant_id = current_setting('app.tenant_id')::int
  );
```

The `execute_sql` tool itself provides configurable access control, read-only mode enforcement, safe SQL parsing, and `statement_timeout` for runaway queries. On top of that, every query gets logged with context, and rate limiting is enforced.

These aren't optional. They're the baseline for running MCP against any real database.

### Getting started with pg-airman-mcp

Setting up pg-airman-mcp takes a few minutes. Add this to your MCP client configuration:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e",
        "AIRMAN_MCP_DATABASE_URL",
        "enterprisedb/pg-airman-mcp",
        "--access-mode=unrestricted"
      ],
      "env": {
        "AIRMAN_MCP_DATABASE_URL": "postgresql://mcp_reader:pass@localhost:5432/mydb"
      }
    }
  }
}
```

Use `--access-mode=restricted` for production. Restart your MCP client. Ask anything.

More details on setup [here](https://github.com/EnterpriseDB/pg-airman-mcp#quick-start)

## The deeper problem: silently wrong queries

Schema discovery solves the obvious failures: wrong table names, missing columns, bad JOINs. There's a another problem that it doesn't touch.

Ask an LLM `**"what's our revenue this month?"**` The SQL executes. The number looks plausible. **BUT it's wrong.**

```sql
-- What the LLM generates
SELECT SUM(o.total_price)
FROM   customer_orders o
WHERE  o.created_at >= '2026-04-01'
```

This runs fine. Returns a number. But it includes cancelled orders, refunded orders, and soft-deleted rows where `is_deleted = true`. The correct query:

```sql
-- What the query should be
SELECT SUM(o.total_price)
FROM   customer_orders o
WHERE  o.created_at >= '2026-04-01'
  AND  o.status != 'cancelled'
  AND  o.is_deleted = false
  AND  o.is_refunded = false
```

That's tribal knowledge. It lives in people's heads, not in the schema. No amount of `pg_catalog` introspection will tell you that `is_deleted = false` is a required filter on every query against `customer_orders`.

### The fix: Custom MCP servers with FastMCP

Encode business logic into MCP tools. The LLM calls a tool that already knows the rules instead of writing SQL from scratch.

```python
from fastmcp import FastMCP
import psycopg2

mcp = FastMCP("revenue-server")

@mcp.tool()
def get_monthly_revenue(month: str, year: int) -> dict:
    """Get actual revenue excluding cancelled,
    refunded, and soft-deleted orders."""
    conn = psycopg2.connect(DB_URL)
    cur = conn.cursor()
    cur.execute("""
        SELECT SUM(o.total_price)
        FROM   customer_orders o
        WHERE  DATE_TRUNC('month', o.created_at)
               = DATE_TRUNC('month', %s::date)
        AND    o.status != 'cancelled'
        AND    o.is_deleted = false
        AND    o.is_refunded = false
    """, [f"{year}-{month}-01"])
    result = cur.fetchone()
    return {"revenue": float(result[0] or 0)}

if __name__ == "__main__":
    mcp.run()
```

The python docstring tells the LLM what the tool does. The **implementation has the business rules** baked in. When the LLM sees a revenue question, it calls `get_monthly_revenue` instead of guessing.

### The hybrid approach

In practice, you need both.

Postgres MCP handles schema, tables, columns, foreign keys. It tells the LLM what exists in the database. It's the discovery layer that prevents structural hallucinations.

Custom MCP encodes business logic, filters, compliance rules, tribal knowledge. It tells the LLM what the data actually means. **It prevents semantic hallucinations.**

How this plays out:

- Legacy schemas with cryptic column names? Postgres MCP discovers them live.
- Status flags, tribal joins? Custom MCP encodes the rules.
- The LLM picks the right tool for the question.

### Why not just...

There are other ways to give LLMs database context. I get asked about other options as well.

| Approach                  | Freshness                    | Scalability    | Verdict                           |
| ------------------------- | ---------------------------- | -------------- | --------------------------------- |
| Paste schema in prompt    | Stale instantly              | Wastes tokens  | Quick hack only                   |
| RAG over docs             | Embeddings drift from schema | Good           | Good for semantics, not structure |
| Fine-tuned model          | Stale on deploy              | Expensive      | Overkill for schema               |
| **MCP (live connection)** | **Always current**           | **Composable** | **Best for structure**            |

- Pasting schema into the prompt works for a toy project. Falls apart when tables change.
- RAG is good for semantic search over documentation but poor for structural metadata that needs to be exact.
- Fine-tuning bakes in schema knowledge that goes stale the moment you deploy.
- MCP queries the live database every time.

## Takeaways

- LLMs hallucinate SQL because they lack live database context. Better prompting doesn't fix this.
- MCP is an open protocol for connecting LLMs to external data. pg-airman-mcp uses `pg_catalog` and `information_schema` to turn Postgres's self-knowledge into LLM context.
- Custom MCP servers built with FastMCP encode business logic and tribal knowledge that schema introspection can't reveal.
- The combination: Postgres MCP for structure, Custom MCP for semantics - is what works for real enterprise databases.

## References

- [Model Context Protocol Specification](https://modelcontextprotocol.io)
- [pg-airman-mcp](https://github.com/EnterpriseDB/pg-airman-mcp)
- [PostgreSQL System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
