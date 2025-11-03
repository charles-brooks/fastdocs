
# DESIGN.md — InstantDocs

This document records **decisions we’re locking in now**, the high‑level file layout, and the basic workflow so we can iterate without getting lost in infra.

---

## 1) Goals & non-goals

**Goals (MVP)**
- Run a **daily loop** that ingests new/changed Zendesk tickets, mines topic clusters, proposes **doc drafts/diffs**, and posts a **Slack digest** for human approval.
- Keep **humans in control**: approved items become **Guide drafts** only. Publishing stays in Team Publishing.
- Keep infra **simple** with AWS Bedrock services; no custom vector DB management.

**Non‑goals**
- Production hardening (multi‑region, HA, autoscaling, SSO, RBAC, etc.)
- Public Help Center metrics; we focus on **CSR signals**.
- Fine‑grained editorial UI (we’ll use Slack + Zendesk for now).

---

## 2) Architecture (high level)

**Control plane:** a single Python process (`run_daily_cycle.py`) orchestrates the pipeline:
1. **Ingest** (Zendesk incremental export; cursors persisted locally)  
2. **Preprocess** (Guardrails PII redaction → ticket summaries)  
3. **Embed & index** (Bedrock embeddings + Bedrock Knowledge Bases vector store)  
4. **Mine gaps** (cluster tickets; compare with KB catalog; score gaps using internal signals)  
5. **Draft** (LLM produces structured doc or diff; include sources)  
6. **Auto‑QA** (PII, grounding, style checks)  
7. **Notify** (Slack digest with Approve / Edit / Reject)  
8. **Publish (draft only)** (Zendesk Guide draft via Translations API for body/title)

**Data stores (minimal)**
- **Vector store**: **Bedrock Knowledge Bases** with quick‑create vector store. Start with **Amazon S3 Vectors** for simplest ops or **OpenSearch Serverless (Vector Engine)** for richer filtering; both are supported.  
- **Raw/derived data**: local `data/` in dev; S3 bucket in team environments (ticket snapshots, summaries, cluster docs).  
- **State**: SQLite (dev) for cursors, job runs, and approvals.

---

## 3) Tech choices we’re locking in

- **LLM & embeddings via Bedrock**  
  - Writer model: configurable per environment; default to an available Bedrock text FM in our region.  
  - Embeddings: `amazon.titan-embed-text` (default) with option to switch to Cohere Embed via config.  
- **Guardrails for Bedrock**  
  - Enable **PII redaction** on both prompts and outputs.  
  - Turn on **contextual grounding** checks for RAG drafts; fail‑closed (send to human if grounding score is poor).  
- **Vector DB**  
  - **Do not self‑host**. Use **Knowledge Bases for Bedrock** to manage chunking, embeddings, and a **managed vector store** (quick‑create). Initial target: **S3 Vectors** or **OpenSearch Serverless Vector Engine**.  
- **Zendesk integration**  
  - **Incremental Export** (cursor‑based) for tickets.  
  - **Articles API** for metadata, **Translations API** for title/body updates (drafts only).  
  - Use **Team Publishing** in product for review/approval and **Verification rules** for freshness.  
- **Human‑in‑the‑loop**  
  - Slack (`chat.scheduleMessage`) for daily digest; **Block Kit** buttons for Approve / Edit / Reject.  
  - Telegram optional later (avoid PII in TG).  
- **Runtime**  
  - Single worker / cron job; retries with exponential backoff. Respect Zendesk rate limits.

**Why Bedrock KB vs rolling our own vector DB?**  
- It removes the “build/manage/secure” overhead for ingestion/chunking/embedding and supports **multiple vector stores** behind one API. We can switch from S3 Vectors to OpenSearch Serverless or partner stores later without refactoring core code.

---

## 4) File tree (starting point)

```

instantdocs/
├─ README.md
├─ DESIGN.md
├─ requirements.txt
├─ .env.example
├─ scripts/
│  ├─ backfill_ingest.py          # one-time historical ingest (paged; cursor-based)
│  └─ run_daily_cycle.py          # end-to-end daily loop
└─ src/instantdocs/
├─ **init**.py
├─ config.py                   # env & constants (models, KB id, guardrail id…)
├─ logging.py
├─ storage/
│  ├─ state.py                 # sqlite cursors, job runs, approvals
│  └─ blob_store.py            # local fs or S3
├─ zendesk/
│  ├─ client.py                # tickets/articles/translations APIs
│  ├─ ingest.py                # incremental export reader + normalizer
│  └─ helpcenter.py            # list articles, create draft, update translation
├─ pipeline/
│  ├─ redact.py                # Bedrock Guardrails prefilter (PII)
│  ├─ summarize.py             # per-ticket short abstracts
│  ├─ embed.py                 # Bedrock embeddings
│  ├─ index.py                 # push to Bedrock KB vector store
│  ├─ cluster.py               # topic clustering + exemplars
│  ├─ signals.py               # join with KB index + agent signals (linked/flagged/created)
│  ├─ rank.py                  # gap score (freq × friction × lack-of-coverage)
│  ├─ retrieve.py              # KB + ticket exemplars for draft context
│  ├─ draft.py                 # structured doc/diff generation
│  └─ qc.py                    # grounding + style + PII checks
├─ hitl/
│  ├─ slack.py                 # digest + actions
│  └─ decisions.py             # approve/edit/reject → next steps
└─ publish/
└─ guide.py                 # create/update Zendesk Guide drafts via Translations

```

---

## 5) Workflows (CLI)

**Historical backfill**
```

python scripts/backfill_ingest.py 
--since 2024-01-01 
--resource tickets

```

**Daily cycle**
```

python scripts/run_daily_cycle.py 
--limit-clusters 10 
--post-slack true

```

**Approve from Slack**
- “Approve” → create/update a **Guide draft translation**.  
- “Edit” → re‑prompt writer with reviewer notes.  
- “Reject” → archive candidate; feedback logged to improve ranking.

---

## 6) Data model (lightweight)

- `TicketSummary`: `{ticket_id, summary, product_area, locale, ts}`  
- `Cluster`: `{cluster_id, centroid_embedding, top_terms, size, exemplars[]}`  
- `CoverageSignals`: `{cluster_id, linked_count, flagged_count, created_count, macro_hits}`  
- `GapCandidate`: `{cluster_id, kb_match_ids[], gap_score, proposed_doc_type}`  
- `Draft`: `{draft_id, cluster_id, title, body_html, sources[], status}`  
- `Decision`: `{draft_id, reviewer, decision, notes, ts}`

---

## 7) Guardrails & quality

- **PII redaction** on both **prompts and outputs** before storage or Slack.  
- **Contextual grounding** check (RAG) → if low confidence, route to “needs clarifications”.  
- Style rules baked into the prompt (voice, headings, steps, warnings).  
- Only **drafts** are created via API—**publishing remains human**.

---

## 8) Zendesk specifics (MVP)

- **Incremental Export** (cursor‑based) for tickets. Persist `after` cursor; backoff on HTTP 429.  
- **Articles**: list, create drafts; **Translations API** to update **title/body** (metadata updates via Articles API).  
- **Team Publishing**: reviewers approve and publish; **Verification rules** send periodic reminders.  
- Watch per‑endpoint **rate limits** (incremental export: 10 req/min by default).

---

## 9) Configuration

`.env.example`
```

AWS_REGION=us-east-1
BEDROCK_MODEL_ID=provider.model-id:variant
BEDROCK_EMBEDDING_MODEL_ID=amazon.titan-embed-text-v2:0
BEDROCK_GUARDRAIL_ID=grd-xxxx
KB_ID=kb-xxxx
KB_VECTOR_STORE=s3   # or opensearch
ZENDESK_SUBDOMAIN=acme
ZENDESK_EMAIL=[ops@acme.com](mailto:ops@acme.com)
ZENDESK_API_TOKEN=...
ZENDESK_LOCALE=en-us
SLACK_BOT_TOKEN=xoxb-...
SLACK_CHANNEL_ID=C123...

```

---

## 10) Open questions / next steps

- Do we want **multilingual** draft support now (switch to multilingual embeddings)?  
- Should we store **ground‑truth fixes** (e.g., macro steps) as reusable snippets?  
- When do we allow **safe auto‑updates** (typos/link rot) without human approval?
