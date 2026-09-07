# geo-doc-extraction-agent

[![CI](https://github.com/santiago-ven-ai/geo-doc-extraction-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/santiago-ven-ai/geo-doc-extraction-agent/actions/workflows/ci.yml)
[![Coverage](https://img.shields.io/badge/coverage-53%25-yellow)](https://github.com/santiago-ven-ai/geo-doc-extraction-agent/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A confidence-gated extraction agent that turns unstructured geological survey reports into a queryable, schema-validated dataset.

## Pitch Card

**Problem** — Mining exploration teams hold decades of drilling reports as unstructured PDFs. Extracting mineral, depth, and coordinate data by hand blocks every downstream analysis.

**Solution** — An extraction agent: retrieve relevant passages, extract to a domain schema, validate units and coordinate ranges, and retry only when extraction confidence is low — instead of blindly re-running or blindly trusting a single pass.

**Impact** — 15/15 reports extracted with **100% field-level precision** and 100% schema conformance, verified with real MiniMax M3 calls (not the fake provider) on a 15-report synthetic set — see "Measured" below for why this task scores higher than `agentic-claims-copilot`'s retrieval task, and how the confidence-gate's failure path was verified separately.

**Stack** — Python 3 · PySpark · FastAPI · Go/Gin · Pinecone (Chroma in dev) · MiniMax M3 (LLM) · AWS (Step Functions, Lambda, S3, DynamoDB) via MiniStack

---

## Architecture

```
  synthetic/public geological reports
             |
             v
  src/ingestion/gateway (Go/Gin intake gateway)
    validate, dedupe by content hash
             |
             v
        S3 (geo-docs)
             |
             v
  src/ingestion/index_docs.py
    noisy-text chunking + embeddings
             |
             v
     Pinecone / Chroma
             |
             v
  src/orchestration/statemachine.py drives the loop below, calling into
  src/models/extraction_agent.py for each step (extraction stays in
  Python here — an LLM call doesn't fit a bare Lambda runtime):
             |
             v
       +-----------+
   +-->|   Plan     |
   |   +-----+-----+
   |         v
   |   +-----------+
   |   |  Extract   |  mineral, depth_m, lat/lon, assay grade
   |   +-----+-----+
   |         v
   |   +-----------+
   |   |  Validate  |  Pydantic domain schema (units, CRS bounds)
   |   +-----+-----+
   |         v
   |   Gate (Lambda): src/orchestration/lambdas/check_attempt.py
   |     atomic retry-attempt counter
   |         |
   |     +---+--------------------+
   |     v                        v
   |  confidence high        confidence low
   |     |                        |
   |     v                   retry left? --no--> fail explicitly, reason recorded
   |  DynamoDB                    |
   |  (structured records)       yes
   |                              |
   +------------------------------+

  src/transformation/resolve.py (PySpark)
    cross-document entity resolution
             |
             v
        S3 Parquet
             |
             v
  src/utils/warehouse.py :: DuckDB (Redshift stand-in)
             |
             v
  src/serving/api.py :: FastAPI --> /search  /extract/{doc_id}
```

See `docs/architecture.md` for the diagram.

## Why Go here

The intake gateway validates, rate-limits, and dedupes by content hash before a document reaches OCR/embedding — expensive steps worth gating behind a fast, stateless check. No business logic lives here, only the gate.

**Honesty note:** production-grade Python core; Go is a bounded intake worker, not evidence of Go platform seniority.

## Measured in this repo

| Metric | Value | How it's measured |
|---|---|---|
| Field-level precision (15 reports × 6 fields = 90 fields) | **1.0** (90/90 correct) | `LLM_PROVIDER=minimax python3 scripts/eval.py` |
| Schema conformance rate | **1.0** (15/15) | same eval run |
| Avg extraction attempts per report | 1.0 (no retries needed — clean synthetic text) | same eval run |
| Confidence-gate failure path (provider that never produces valid JSON) | **fails gracefully after exactly 2 attempts**, reports a reason, never a silently-invalid record | `pytest tests/unit/test_extraction.py::test_confidence_gate_fails_gracefully_not_silently` |
| Domain schema catches out-of-region coordinates | **verified**: a lat/lon outside the survey bounding box is rejected even though it type-checks | `pytest tests/unit/test_extraction.py::test_coordinate_outside_survey_region_rejected` |
| Intake gateway dedup (Go/Gin + DynamoDB conditional write) | **verified live**: identical content under a different `report_id` → 409 | manual curl test, see BUILD_GUIDE |

**Why this scores higher than `agentic-claims-copilot`'s retrieval numbers:** this is direct fact extraction from explicit, unambiguous text ("intersected 1.19 g/t Nickel at a depth of 77m") — a much easier task for an LLM than semantic retrieval over paraphrased claim language against a small local embedding model. A perfect score here and a modest 0.17 there are both honest results of tasks with genuinely different difficulty, not inconsistent quality.

## Modeled business impact (synthetic data — assumptions documented)

| Assumption | Source | Modeled outcome |
|---|---|---|
| Geologist hours/report spent on manual data entry, reports processed/month | TODO — cite in `docs/impact-model.md` | TODO |

## Emulated vs. real

| Component | Dev (this repo) | Production would use | Fidelity |
|---|---|---|---|
| S3 / Lambda / DynamoDB | [MiniStack](https://ministack.org) (free, MIT, no account) | AWS | High |
| Step Functions | MiniStack (full ASL interpreter) | AWS | Medium-High |
| AWS CLI v2 | Real `aws` CLI against MiniStack (`AWS_ENDPOINT_URL`) — see `docs/RUNBOOK.md` §2 | AWS CLI v2 | High |
| Secrets Manager | MiniStack — `MINIMAX_API_KEY`/`PINECONE_API_KEY` are stored via `scripts/secrets_setup.py` and read through `utils/secrets.get_secret()`, which falls back to the env var only if the secret isn't there | AWS Secrets Manager | High — `create-secret`/`get-secret-value` round-trip correctly; see `docs/RUNBOOK.md` §5 ex. 4 |
| Redshift | **DuckDB**, reading the resolved-entity Parquet directly from S3 | Redshift Serverless | Medium — no MPP distribution; real DDL in `sql/redshift/` |
| Vector store | Chroma (`VECTOR_BACKEND=chroma`) or real Pinecone (`=pinecone`) | Pinecone | High — same interface |
| LLM | Deterministic fake (`LLM_PROVIDER=fake`) | Real [MiniMax M3](https://minimax-ai.chat/docs/api/) (`=minimax`) | Precision/recall in the README are measured with the real provider |

## Three non-tutorial challenges

1. **Noisy post-OCR chunking** — boundary detection that survives broken line breaks and tables without destroying retrieval quality.
2. **Confidence-gated retry, not blind retry** — the confidence threshold is calibrated against the labeled set, not picked arbitrarily.
3. **Domain schema validation** — units (ft vs. m), coordinate reference systems, plausible value ranges. A record that passes the schema but places a mine in the ocean is a bug, not a pass.

## Installation

```bash
git clone https://github.com/santiago-ven-ai/geo-doc-extraction-agent.git
cd geo-doc-extraction-agent
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt   # app deps + lint/type/security tooling
```

## Usage — Demo (3 minutes)

```bash
source env.sh
make demo   # 15 synthetic geological reports (docs/RUNBOOK.md)
pytest tests/unit/test_extraction.py
make query
```

## Testing

```bash
make test                     # unit + BDD (pytest-bdd), against real MiniStack + Go gateway
make e2e                      # full pipeline, emits benchmarks/quality-report.json
.venv/bin/pre-commit run --all-files   # ruff, mypy, gofmt, whitespace/EOF checks
```

CI (`.github/workflows/ci.yml`) runs the same suite on every push, plus an isolated `security` job (`pip-audit` with two chromadb CVEs explicitly suppressed — see the job's own comment — and `gosec` on the Go gateway) and a coverage gate that fails the build under the threshold on the badge above.

## What this is NOT

Not a "PDF-to-text" tutorial. The domain schema, the confidence gate, and the labeled evaluation set are what make this engineering.

## Build it yourself

See [`docs/RUNBOOK.md`](docs/RUNBOOK.md) to run the flow, or [`docs/BUILD_GUIDE.md`](docs/BUILD_GUIDE.md) to build from scratch.

## Contributing

Solo-maintained portfolio/demo repo — not actively seeking external contributions, but issues and questions are welcome via [GitHub Issues](https://github.com/santiago-ven-ai/geo-doc-extraction-agent/issues). See [`CODEOWNERS`](.github/CODEOWNERS) and [`SECURITY.md`](SECURITY.md) for how reports are handled.

## License

[MIT](LICENSE) © santiago-ven-ai
