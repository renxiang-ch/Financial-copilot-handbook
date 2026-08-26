# 01 · Environment Setup

Covers: `pyproject.toml`, `uv.lock`, `.env.example`, `config.py`,
`storage/schema.py`, `pipeline/seed.py`, `start.ps1`.

## 1.1 Install

```bash
git clone <repo-A-url>
cd <repo-A-name>
uv sync
```

`uv` reads `pyproject.toml` (the human-declared dependency list) and
`uv.lock` (every dependency pinned to an exact version + hash) and builds a
`.venv`. Always invoke project commands as `uv run --active python -m ...` —
never call `.venv\Scripts\python` directly. The package lives under `src/`
(a "src layout"), and only `uv run` puts it on the path correctly; a direct
interpreter call fails to find the `copilot` package.

PostgreSQL 17+ with the `pgvector` extension is required, on either OS:

- **Mac**: Postgres.app ships pgvector 0.8.2 built in — no separate install
  step.
- **Windows**: install PostgreSQL via the EDB installer, then build
  `pgvector` from source with Visual Studio Build Tools and copy the
  resulting DLL into PostgreSQL's `lib/` directory — there is no prebuilt
  Windows binary for it as of this writing. This is a real build step, not
  a package-manager one-liner; budget time for it on a first Windows setup.

## 1.2 Configure

```bash
cp .env.example .env
```

| Variable | Controls |
|---|---|
| `OPENAI_API_KEY` | The agent's default model (`gpt-4o-mini`) and the eval harness's own model calls. Required for anything that calls the LLM. |
| `ANTHROPIC_API_KEY` | Only used if you explicitly pass a `claude*` model id to the agent (`agent.ask(question, model="claude-...")`); the default path never touches it. |
| `DATABASE_URL` | `postgresql://user:password@host:5432/financial_copilot` — everything downstream in this chapter depends on this pointing at a real, reachable Postgres instance with `pgvector` installed. |
| `LANGFUSE_*` | Optional observability; the app runs without it. |

## 1.3 Create the database and load the snapshot

```bash
createdb financial_copilot
psql -d financial_copilot -c "CREATE EXTENSION IF NOT EXISTS vector;"
uv run --active python -m copilot.pipeline.seed --load
```

**Why this loads a snapshot rather than re-running ingestion from EDGAR**
(from `seed.py`'s own module docstring, reproduced here rather than
re-derived, because the reasoning is specific and easy to flatten into a
vaguer claim than what's actually true): re-running the ingestion pipeline
does **not** reconstruct this database, and claiming it does would be a
reproducibility claim that fails quietly. Two concrete reasons: `supply_edges`
carries corrections applied by hand in a `psql` session (Qorvo's
FY2018/FY2019 percentages, Skyworks' FY2016/FY2017 threshold classification,
five Jabil citation fixes, a three-row Huawei entity merge) that no script
contains; and a text-extraction fix (`ix:nonfraction` handling) landed
*after* the last extraction run and would change the candidate text blocks
for 32 of 54 filings if extraction were re-run today — deliberately not
re-run, because the newly-visible text is of mixed quality and re-extracting
risks introducing wrong edges, not just recovering missing ones. The honest
artifact is therefore the snapshot every published result was actually
measured against, tied to those results by a shared `db_fingerprint`.

Expected output ends with a fingerprint line matching
`data/seed/manifest.json`'s `fingerprint` object — see Appendix B for the
exact current values. If it does not match, stop here: nothing downstream
will reproduce.

**Embeddings are deliberately not in the seed.** They are ~24MB of float32
vectors that `embed_chunks` (next step) regenerates deterministically from
the chunk text using a local model — no API key, no network call. Chunk
*text* is in the seed, since it is the actual evidence and is not
reconstructable from anything else without re-fetching every filing from
EDGAR. Immediately after loading the seed, `embedding` is `NULL` on every
row and retrieval runs BM25-only — a visible, documented gap, not a silent
one, until the next step runs.

## 1.4 Generate embeddings

```bash
uv run --active python -m copilot.pipeline.embed_chunks
```

Runs locally (`BAAI/bge-small-en-v1.5`, 384 dimensions, no API key), CPU is
fine for one-off generation over 16K chunks — budget on the order of tens of
minutes on a laptop CPU, faster with a GPU. Until this completes, dense
retrieval returns nothing and the hybrid retriever silently degrades to
BM25-only; there is no error, just a weaker `retrieve_text`, so don't skip
this step and assume everything works because nothing crashed.

## 1.5 Start

```powershell
.\start.ps1          # Windows: kills anything already on 8000/8501, then
                      # starts FastAPI on :8000 and Streamlit on :8501
```

No Mac-equivalent launch script ships in this repo as of this writing —
on Mac, start the two processes directly: `uv run --active uvicorn
copilot.api:app --port 8000` and, in a second terminal, `uv run --active
streamlit run frontend.py`. See ch.07 for the specific Windows pitfall
`start.ps1`'s port-killing step exists to prevent (a stale Streamlit
process surviving a restart and corrupting its own internal page table).

## 1.6 Verify

```bash
PYTHONUTF8=1 uv run --active python -m copilot.eval.harness --out /tmp/check.json
```

(On Windows PowerShell: `$env:PYTHONUTF8="1"; uv run --active python -m copilot.eval.harness --out /tmp/check.json` — the `PYTHONUTF8` flag is required to avoid `cp1252` encoding errors on a Windows terminal; the harness prints non-ASCII characters in some trace output.)

Expect Tier-1/Tier-2/input-fetch/refusal accuracy at 100% and overall
accuracy in the high 80s — see Appendix B for the exact current numbers and,
importantly, which of these figures is a fixed target versus a noise band.
A retrieval-accuracy figure noticeably below its historical band (25–62.5%)
is not automatically a broken setup — read ch.05 §5.1 and §5.3 before
concluding something is wrong.
