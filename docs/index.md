# filing-qa

**A research prototype for question answering over financial filings.** Multimodal RAG with
hybrid retrieval and numeric grounding checks, measured on a 119-question golden set. It has not
been run as a real deployment, and chart extraction has not been validated. The
[README status section](https://github.com/Mattral/filing-qa#status) lists what is and is not
measured.

---

## System Architecture

![Architecture Pipeline](assets/architecture_pipeline.png)

*Complete ingestion and query pipeline. Every component implements an ABC — switch any layer via a single config value.*

---

## Approach

| Problem | Approach | Status |
|---|---|---|
| Some data appears only in charts | A vision LLM (GPT-4o / Gemini / Qwen2-VL / local vLLM, with a fallback chain) describes each chart image at ingest | Implemented, not evaluated. Extraction accuracy has not been measured |
| Exact financial figures miss semantic search | Hybrid RRF: dense embeddings + BM25 keyword search fused with Reciprocal Rank Fusion | Implemented, tested, and measured offline (evidence hit@5 0.71) |
| LLMs can state numbers their sources do not contain | Numeric grounding guardrail flags numbers in the answer that are absent from the retrieved text | Implemented, tested by unit tests. Not run on the golden set |
| One failing provider stops a stage | Vision provider chain, parser fallback to `pypdf`, generator fallback model on 429/503 | Implemented. Vision chain and parser fallback are tested with mocks; the generator fallback is not tested |
| Data must not leave the network | Local/open-source configuration: vLLM + BAAI/bge + pgvector | Implemented, not evaluated. No fully local run has been recorded |

---

## Quickstart

```bash
git clone https://github.com/Mattral/filing-qa
cd filing-qa
cp .env.example .env
docker compose up -d
curl -X POST http://localhost:8000/api/v1/ingest -F "file=@your_10k.pdf"
curl -X POST http://localhost:8000/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What was gross margin in Q3 2023?"}'
```

See [Quickstart → Docker](quickstart/docker.md) for the full walkthrough.

---

## Evaluation

![Retrieval quality, measured](assets/retrieval_measured.png)

![Answer quality, measured](assets/answer_quality_measured.png)

* **Golden dataset:** 119 QA pairs over 16 public filings from 10 companies, every figure
  traced to its source passage. See the
  [dataset README](https://github.com/Mattral/filing-qa/blob/main/evals/golden_datasets/README.md).
* **Retrieval, measured offline on the real filings:** hybrid RRF puts an answer-bearing chunk in
  the top 5 for 71% of questions at the default 4,000-character chunks, the same as BM25 alone.
  With 1,000-character chunks it reaches 53% versus 42% for BM25.
* **Answer quality (all 119 questions, `gemini-3.8-flash`, one run, deterministic scoring with no
  LLM judge):** the asked-for figure was stated in 68% of answers (84% when retrieval found the
  supporting text). An automated check flagged 6 answers that stated a number absent from their
  context. On review by hand all six were correct derivations. The check cannot catch a real
  number used for the wrong thing. Latency under load and cost are not yet measured, and earlier
  figures were withdrawn. See
  [BENCHMARK_RESULTS.md](https://github.com/Mattral/filing-qa/blob/main/BENCHMARK_RESULTS.md).

---

## SLO Alerting

![SLO Burn Rate](assets/slo_burn_rate.png)

*Design diagram, not live data: it illustrates the configured alert rules and is not taken from a
running deployment. Multi-window multi-burn-rate alerting (Google SRE Workbook), with four alert
tiers routed to PagerDuty/OpsGenie.*

---

## Provider Support

![Provider Matrix](assets/provider_matrix.png)

*Every model-facing layer is independently configurable. Switch with one env var — no code changes.*

---

## Key Numbers

| Metric | Value |
|---|---|
| Golden eval samples | 119 |
| Architecture Decision Records | 8 |
| Prometheus metrics | 17 |
| Grafana dashboards | 2 |
| K8s manifests | 15 |
| Vector store backends | DeepLake, pgvector, Qdrant (all real implementations) |
| Providers supported | 4 LLMs · 4 vision · 4 embedders · 3 vector stores · 3 cloud connectors |
