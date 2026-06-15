# Conversational Data Analyst Agent

A Slack bot that lets business users ask questions in plain English. The bot turns the question into SQL, runs it safely against a Postgres database, and replies with a formatted table and a chart.

Built with n8n, OpenAI gpt-4o-mini, Docker, and Slack.

## Demo

User asks in Slack: *"Top 5 customers by total revenue"*
→ Bot replies in ~5 seconds with a table and a bar chart.

## Architecture

```
User (Slack DM)
    ↓
Slack Events API → ngrok tunnel
    ↓
n8n Webhook
    ↓
URL Verification check  →  Respond to Slack handshake
    ↓
Acknowledge Slack (< 3s response)
    ↓
Filter (drop bots, system events)
    ↓
Extract Question
    ↓
OpenAI gpt-4o-mini — SQL Generator (schema in system prompt, JSON output)
    ↓
SQL Validator (JavaScript: allowlist SELECT/WITH, block DDL/DML)
    ↓
Postgres — Execute Query
    ↓
    ├─ Success → Format Results → Build Chart (QuickChart) → Send to Slack
    └─ Failure → SQL Fixer (LLM with error context) → retry, max 3 attempts
```

## Key Design Decisions

1. **The LLM is an untrusted translator, not a database client.** Every query passes through a code-level validator before reaching Postgres. SQL injection through the natural language layer is blocked by an allowlist of SELECT/WITH and word-boundary regex against DDL/DML keywords.

2. **Self-correction loop.** Failed queries feed back to the LLM with the error message — the error contains exactly the information the LLM was missing. Usually fixed on the first retry. Capped at 3 attempts.

3. **Schema injection at prompt time.** Full Northwind schema in system prompt. Works for ~50 tables; beyond that, switch to RAG over schema (embed table descriptions, retrieve top-K per query).

4. **Defense in depth.** System prompt restricts to SELECT, validator enforces it in code, production would add a read-only DB user as a third independent layer.

5. **Auto-charting.** QuickChart generates inline bar charts when results have 3+ rows with numeric data.

6. **JSON output enforcement.** OpenAI configured with `jsonOutput: true` so model returns structured `{sql, explanation}` rather than free text.

## Stack

| Component | Purpose |
|---|---|
| n8n (Docker) | Workflow orchestration + visual debugging |
| Postgres 16 (Docker) | Northwind sample dataset |
| OpenAI gpt-4o-mini | SQL generation + self-correction |
| Slack Events API | User interface |
| ngrok | Public tunnel for local dev |
| QuickChart.io | Server-side chart rendering |

## Setup

### Prerequisites
- Docker Desktop
- ngrok account (free tier)
- OpenAI API key
- Slack workspace with custom app permissions

### 1. Clone and configure

```bash
git clone https://github.com/YOUR-USERNAME/conversational-data-analyst.git
cd conversational-data-analyst
cp .env.example .env  # fill in your keys
```

### 2. Get Northwind data

```bash
cd init
curl -o northwind.sql https://raw.githubusercontent.com/pthom/northwind_psql/master/northwind.sql
cd ..
```

### 3. Start services

```bash
docker-compose up -d
```

### 4. Verify Postgres loaded

```bash
docker exec -it cgi-postgres psql -U admin -d analytics -c "SELECT COUNT(*) FROM customers;"
# Should return 91
```

### 5. Start ngrok tunnel

```bash
ngrok http 5678
```

Copy the `https://...ngrok-free.dev` URL.

### 6. Slack app setup

1. Create app at [api.slack.com/apps](https://api.slack.com/apps) → From scratch
2. Add Bot Token Scopes: `app_mentions:read`, `chat:write`, `im:history`, `im:read`, `im:write`, `files:write`
3. Install to workspace → copy Bot User OAuth Token (xoxb-...)
4. Enable Events → set Request URL to `https://YOUR-NGROK.ngrok-free.dev/webhook/slack-events`
5. Subscribe to bot events: `app_mention`, `message.im`
6. Reinstall app after permission change

### 7. n8n setup

1. Open `http://localhost:5678`
2. Create credentials:
   - **OpenAI**: paste your API key
   - **Postgres**: host `cgi-postgres`, port 5432, user `admin`, db `analytics`, password `admin`, SSL disabled
   - **Slack Header Auth**: name `Authorization`, value `Bearer xoxb-YOUR-TOKEN`
3. Import `workflows/sql-agent.json`
4. Wire credentials to nodes (OpenAI nodes, Postgres node, Slack HTTP Request node)
5. Activate the workflow

### 8. Test

DM the bot in Slack: `how many customers do we have?` → expect `91`

## Demo Queries

| # | Query | What it tests |
|---|---|---|
| 1 | How many customers do we have? | Simple count |
| 2 | Top 5 customers by total revenue | Joins + aggregation + chart |
| 3 | Show me revenue by month for 1997 | Time series + chart |
| 4 | Which product categories sell best in Germany? | Filter + group + multi-join |
| 5 | Drop the customers table | Security demo — bot refuses |

## What I'd Build Next

- **Eval harness** — golden Q&A pairs, automated result comparison, regression test on every prompt change
- **Schema retrieval** — RAG over schema for databases with 100+ tables
- **Router step** — classify questions before SQL generation (answerable / ambiguous / off-topic)
- **Semantic layer** — dbt metrics so "revenue" means the same thing across all queries
- **Read-only DB user** — defense in depth
- **Conversation memory** — multi-turn analysis ("now break that down by month")
- **Cost routing** — small fine-tuned model for simple queries, gpt-4o-mini for complex

## Project Structure

```
conversational-data-analyst/
├── docker-compose.yml          # Postgres + n8n services
├── .env.example                # Environment template
├── .gitignore                  # Excludes pgdata, n8n_data, secrets
├── README.md                   # This file
├── init/
│   └── README.md               # How to fetch Northwind
└── workflows/
    └── sql-agent.json          # n8n workflow export
```

## License

MIT
