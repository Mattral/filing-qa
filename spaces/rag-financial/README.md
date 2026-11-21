---
title: Multimodal Financial RAG
emoji: 🏦
colorFrom: indigo
colorTo: blue
sdk: gradio
sdk_version: 4.44.0
app_file: app.py
pinned: true
license: mit
short_description: Research prototype RAG for financial filings
tags:
  - finance
  - rag
  - nlp
  - document-question-answering
  - gradio
  - multimodal
  - openai
  - gemini
---

# 🏦 Multimodal Financial RAG

**A research prototype for question answering over financial filings** — chart description · hybrid RRF retrieval · numeric grounding checks · source citations

This Space is a demo of the research prototype at
[github.com/Mattral/filing-qa](https://github.com/Mattral/filing-qa).

## Features

- **👁️ Vision chart extraction** — GPT-4o / Gemini 2.5 Flash describe charts and graphs into searchable text
- **⚡ Hybrid RRF retrieval** — dense (BAAI/bge-small-en-v1.5) + BM25 fused with Reciprocal Rank Fusion (k=60)
- **🔢 Numeric grounding check** — flags any number in the answer that does not appear in the retrieved context. It cannot catch a real number used for the wrong thing, and correct derived figures are flagged too
- **🔒 PII and injection pattern checks** — after the answer is generated, the query is scanned with regex patterns for identifiers (SSN, IBAN, CUSIP, ISIN, card, email, phone, account) and for known prompt-injection phrases. Matches are flagged in the Guardrails tab; the query is not changed before it reaches the model. How well the patterns work has not been measured
- **📍 Citations** — the Sources tab lists the document and page of each retrieved chunk. Page numbers inside the answer text are written by the model and are not checked; in the main pipeline they are unreliable when chunks lack page metadata
- **🔬 Full pipeline transparency** — see every retrieval score, RRF weight, generation cost

## Models (v2.0 — current)

| Provider | Text generation | Vision |
|---|---|---|
| Google Gemini | gemini-2.5-flash (default), gemini-2.5-pro | gemini-2.5-flash |
| OpenAI | gpt-4o-mini, gpt-4o | gpt-4o |

## How to Use

1. Enter your API key (Gemini free tier at [aistudio.google.com](https://aistudio.google.com))
2. Upload a 10-K, 10-Q, or earnings release PDF
3. Click **Process Document**
4. Ask a question

## Architecture

```
PDF → pdfplumber (text + tables) + Vision LLM (charts)
    → Semantic chunking (≤800 chars, 100-char overlap, paragraph-boundary split)
    → BAAI/bge-small-en-v1.5 embeddings → FAISS IndexFlatIP
    → BM25Okapi keyword index
    → RRF fusion: 0.7/(60+dense_rank+1) + 0.3/(60+bm25_rank+1)
    → Top-k chunks → GPT-4o-mini / Gemini 2.5 Flash generation
    → Checks on the result: injection patterns → PII patterns → numeric grounding (flag only)
    → Answer + cited sources
```

Mirrors the production pipeline in
[`src/filing_qa/`](https://github.com/Mattral/filing-qa/tree/main/src/filing_qa).

## Privacy

API keys are never stored. Uploaded PDFs are processed in-memory and not persisted.

## License

MIT
