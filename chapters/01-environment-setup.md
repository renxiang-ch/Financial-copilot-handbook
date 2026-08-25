# 01 · Environment Setup

Covers: `pyproject.toml`, `uv.lock`, `.env.example`, `config.py`,
`storage/schema.py`, `pipeline/seed.py`, `start.ps1`.

## 1.1 Install

```bash
git clone <repo-A> && cd <repo-A> && git checkout v1.0-teaching
uv sync
```

<!-- TODO: PostgreSQL 17+ with pgvector — installation per OS. Mac: Postgres.app.
     Windows: EDB installer + pgvector built from source, see repo A's CLAUDE.md
     "Device Configuration" section for the exact Windows build steps if needed. -->

## 1.2 Configure

```bash
cp .env.example .env
```

<!-- TODO: table of the three variables (OPENAI_API_KEY, DATABASE_URL, API_KEY)
     and what each controls -->

## 1.3 Create the database and load the snapshot

```bash
createdb financial_copilot
psql -d financial_copilot -c "CREATE EXTENSION IF NOT EXISTS vector;"
uv run --active python -m copilot.pipeline.seed --load
```

<!-- TODO: explain WHY this loads a snapshot rather than re-running ingestion
     from EDGAR — see pipeline/seed.py's module docstring, copy the reasoning,
     don't re-derive it -->

Expected output ends with a fingerprint line. It must read:

```
<TODO: paste the exact fingerprint dict from data/seed/manifest.json at tag time>
```

If it does not match, stop here — nothing downstream will reproduce.

## 1.4 Generate embeddings

```bash
uv run --active python -m copilot.pipeline.embed_chunks
```

<!-- TODO: expected runtime on CPU, what "BM25-only" degradation looks like if
     this step is skipped -->

## 1.5 Start

```powershell
.\start.ps1          # Windows: FastAPI on :8000, Streamlit on :8501
```

<!-- TODO: Mac equivalent if repo A ships one; Stop-Port pitfall goes in ch.07 -->

## 1.6 Verify

```bash
$env:PYTHONUTF8="1"; uv run --active python -m copilot.eval.harness --out /tmp/check.json
```

<!-- TODO: paste expected accuracy numbers, point to appendix b for the full table -->
