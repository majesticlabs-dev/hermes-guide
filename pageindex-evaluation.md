# PageIndex (VectifyAI/PageIndex) — Evaluation Guide

**Date:** 2026-04-22 · **License:** MIT · **Repo:** https://github.com/VectifyAI/PageIndex

## What PageIndex Is

PageIndex is a **reasoning-based RAG indexing and retrieval system** for PDF and Markdown documents. It builds a hierarchical tree index (a "smart table of contents") and then uses LLM reasoning — not vector embeddings — to navigate and retrieve relevant content.

### Two Phases

1. **Indexing** — Transforms documents into a hierarchical tree structure with page ranges, summaries, and node IDs. Heavy LLM usage (20–100+ calls per 50-page PDF).
2. **Retrieval** — Agent-based: an LLM calls `get_document_structure()` → reasons over the tree → calls `get_page_content("5-7")` → synthesizes an answer. No vectors, no chunking.

### What It Is NOT

- Not a general-purpose OCR/parser (uses basic PyPDF2/PyMuPDF)
- Not a vector database or embedding system
- Not a standalone chatbot — it's a library that provides tools for agents

## Architecture Overview

```
PDF → extract pages → detect TOC (LLM)
    → build hierarchical index (LLM) → verify + self-correct (LLM)
    → output: nested JSON tree {title, node_id, start_index, end_index, summary, nodes}
```

```
User question → Agent → get_document_structure() → reason over tree
             → get_page_content("5-7") → synthesize answer
```

Key files: `page_index.py` (core pipeline), `page_index_md.py` (Markdown), `utils.py` (LLM calls via LiteLLM), `client.py` (workspace + doc management), `retrieve.py` (agent tool functions).

## Quality Signals

### Strengths
- **MIT license** — no vendor lock-in
- **Active development** — 284 commits, 9 contributors, last commit April 2026
- **FinanceBench 98.7% accuracy** (from their Mafin 2.5 system)
- **Self-correction loop** — verifies its own TOC mappings and retries
- **Clean client API** — `PageIndexClient` with workspace persistence and lazy loading
- **Multi-format** — PDF and Markdown support
- **Agent-ready** — built-in OpenAI Agents SDK integration

### Concerns
- **High LLM cost** — 20–100+ calls per document for indexing
- **Zero tests** — no test files, no CI
- **Brittle JSON parsing** — regex-based extraction of LLM output
- **No caching** — failed indexing restarts from scratch
- **Prompts embedded in Python** — hard to iterate without code changes
- **No structured logging** — `print()` statements throughout
- **No type hints**

### Maturity: **Early production / Late prototype**

Functional core idea, but lacks tests, logging, error handling, and internal API docs. Suitable for prototyping; needs hardening for production.

## Alternatives Comparison

| Feature | PageIndex | LlamaIndex | LangChain | Docling |
|---------|-----------|------------|-----------|---------|
| Approach | Reasoning-based tree search | Vector embeddings + chains | Vector embeddings + chains | Document parsing + structure |
| Needs Vector DB | No | Yes | Yes | No |
| Needs Chunking | No | Yes | Yes | No |
| LLM Cost (indexing) | High | Low | Low | None |
| Retrieval Quality | High (reasoning) | Moderate (similarity) | Moderate (similarity) | N/A (parser) |
| Agent Integration | Built-in | Via tools | Via tools | Via tools |
| Maturity | Early | Mature | Mature | Growing |

**Key differentiator:** PageIndex trades higher indexing cost for potentially higher retrieval accuracy via LLM reasoning. Most valuable for complex, long professional documents (legal, financial, regulatory) where semantic similarity search fails.

## Integration Fit with Hermes

### What It Could Do

- **Smart document indexing for KB** — generate tree structures for PDFs, enabling structured navigation without chunking
- **Agent-based retrieval** — integrate `PageIndexClient` tools as Hermes agent tools
- **Markdown indexing** — index existing KB notes with section summaries
- **x-bookmarks enhancement** — index linked PDFs for reasoning-based QA

### Integration Challenges

| Challenge | Severity | Notes |
|-----------|----------|-------|
| LLM cost per document | **High** | 20–100+ LLM calls to index; expensive at scale |
| No incremental updates | Medium | Full re-index on any change |
| No search index | Medium | Still need an agent to query it |
| Event loop conflicts | Low | `asyncio.run()` workaround may conflict with async Hermes |

### Recommended Integration Path

**Short-term:** Use PageIndex to generate tree structures for key KB PDFs. Store JSON alongside documents as navigation aids.

**Medium-term:** Integrate `PageIndexClient` as a Hermes tool for on-demand PDF indexing and reasoning-based retrieval. Requires adding PageIndex as a dependency, creating a Hermes skill wrapper, and budgeting for indexing LLM costs.

**Not recommended for:** Full KB search replacement. Use a hybrid approach — existing tools (`rg`/`qmd`) for search + PageIndex for deep document QA.

### Cost Estimate

~50 documents × 50 LLM calls avg × ~$0.005/call (GPT-4o) = **~$12.50 one-time indexing cost**. Retrieval adds 3–5 LLM calls per question. Manageable for personal/small-team KB.

## Verdict

**GO for targeted integration** — document QA on key PDFs where structured navigation and reasoning-based retrieval adds value. MIT license eliminates vendor risk. Be aware of LLM cost implications at scale.
