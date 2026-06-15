# Architecture Notes

## Why n8n over LangChain/LangGraph

n8n was chosen for this demo because:
- **Visual debugging.** Each node's input and output is inspectable during execution. Failure modes are immediately visible.
- **Speed of iteration.** Prompt changes, retry logic adjustments, and credential rotation don't require redeployment.
- **Workflow-shaped problem.** This system is fundamentally a linear pipeline with one retry loop. n8n is purpose-built for that shape.

For a more complex multi-agent system with shared state, planner-critic loops, or dynamic tool selection, LangGraph would be the appropriate choice.

## Security Layers

| Layer | Mechanism |
|---|---|
| Prompt | System message instructs SELECT-only |
| Validator | Code-level allowlist of SELECT/WITH; word-boundary regex blocks DDL/DML |
| Connection | (Production) Read-only Postgres user |
| Query limit | LIMIT 100 default to bound result size |

## Self-Correction Loop

When Postgres returns an error, the failed query plus error message plus original question are sent to a second LLM call. Most failures are caught on the first retry:

- **Hallucinated columns** — LLM imagines `customers.email` exists; Postgres errors; fixer reads the error and removes/replaces the column.
- **Bad joins** — Wrong join key; error message reveals the actual column names.
- **Type mismatches** — Comparing text to date; fixer adds appropriate casting.

Capped at 3 attempts. Beyond 3, the system gives up and reports the error rather than looping indefinitely.

## Why JSON output

Two reasons:
1. **Robust parsing.** Free-text outputs require regex or LLM-based extraction. JSON output is parsed in one line.
2. **Separation of SQL from explanation.** The model can explain its reasoning without the explanation being mistaken for SQL.

The validator still strips markdown code fences as a defensive layer because some models occasionally wrap JSON output despite explicit instructions.

## Schema Injection vs Retrieval

This demo injects the full Northwind schema (8 tables, ~40 columns) into the system prompt. Total prompt size is comfortable for gpt-4o-mini.

For databases beyond ~30-50 tables:
1. Embed table descriptions + column lists
2. On each question, retrieve top-K most relevant tables
3. Inject only those into the prompt

This is the same RAG pattern applied to schema rather than documents.

## Cost Profile

For typical demo queries:
- SQL Generator: ~500 input tokens (schema) + 100 output tokens = ~$0.0001 per query
- SQL Fixer (when triggered): similar
- 100 queries ≈ $0.01

Production cost optimization would route simple lookups to a fine-tuned small model.

## Latency Budget

End-to-end target: under 10 seconds for happy path.

- Acknowledge Slack: < 100ms
- OpenAI SQL generation: ~2s
- Validation + Postgres execution: < 500ms
- Self-correction (when triggered): adds ~2s per retry
- Chart generation: < 200ms (server-side at QuickChart)
- Slack post: < 500ms

Total: 4-6s on happy path, 8-10s with one retry.

## Failure Modes Encountered During Build

1. **LLM wrapped JSON in markdown code fences** despite explicit instructions. Fix: validator strips ` ```json ` and ` ``` ` fences before parsing.

2. **Initial validator rejected CTE queries** starting with `WITH`. Fix: extended allowlist to include both `SELECT` and `WITH` as valid query starts.

3. **n8n container couldn't reach Postgres via `host.docker.internal`** on some networks. Fix: use the container name `cgi-postgres` since both services share the docker-compose network.

4. **Slack URL verification challenge** required a specific response shape. Fix: dedicated branch in the workflow that echoes the challenge value.

5. **Slack's 3-second timeout** would have caused message retries. Fix: Acknowledge Slack node responds immediately while real processing continues asynchronously.
