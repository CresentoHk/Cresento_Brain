---
title: AI Coach (text_ai_coach)
type: project
tags:
  - project/ai-coach
  - ml
  - python
created: 2026-04-11
status: reference
location: "text_ai_coach/"
---

# 🤖 AI Coach (text_ai_coach)

A Python RAG (retrieval-augmented generation) soccer coach. Pulls relevant content from a vector index and generates coaching suggestions via the Together API (Kimi-K2.5).

> [!info] Reference / not actively shipped
> This is supporting / experimental code, not a production part of the platform yet. Useful as the basis for "ask the coach" features in the apps later.

---

## Stack

- **Python**
- **Together API** — LLM inference (Kimi-K2.5)
- **Tavily** — web fetcher for fresh content
- A local **vector index** for retrieval

---

## Layout

```
text_ai_coach/
├── ai_coach.py          (main entry — RAG loop)
├── tavily_fetcher.py    (web content fetching)
├── vector_index.py      (build / query the vector index)
├── vector_index/        (persisted index files)
├── scraped_content/     (cached fetched pages)
└── my_topics.txt        (topics to scrape / index)
```

---

## High level

1. `tavily_fetcher.py` pulls relevant soccer-coaching content
2. `vector_index.py` chunks + embeds it into a local index
3. `ai_coach.py` takes a question, retrieves matching chunks, builds a prompt, calls Together API, returns the answer

This is the standard RAG pattern — useful as a template if/when this gets folded into the apps.

---

## Related

- [[Vision System (Veo Pipeline)]] — sibling Python project
- [[Experimental Implementation]] — IMU data exploration
