---
title: "Why Single-Index RAG Struggles Across Multiple Data Sources (And How to Fix It)"
description: "When a question spans policy docs, SQL tables, and JSON logs, a single vector index often drops key facts. On a 60-query synthetic benchmark, splitting retrieval and reserving prompt slots lifted recall from 48% to 93%."
tags: [rag, search, llm]
featured: false
---

Most introductory RAG tutorials start the same way: take a folder of documents, split them into 500-token chunks, embed them into a vector database, and run cosine similarity.

That works well when all your answers live in prose articles. It starts to struggle when an application needs to combine different types of data.

In many software applications, information isn't stored in one folder of text. It is spread across **help articles**, **relational database tables**, and **JSON event logs**. Answering a straightforward user question often requires pulling a piece of context from all three at the same time.

---

## 1. A Toy Example That Breaks Single-Index RAG

Consider a hypothetical retail support question:

> *"Can I still return the headphones from order `#ORD-402` if I already opened the box?"*

It sounds simple, but the facts needed to answer it live in three different places:

```text
   "Can I still return the headphones from order #ORD-402 if I opened the box?"
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
 [Unstructured Docs]        [Structured SQL DB]        [Semi-Structured JSON]
 Store Return Policy        Orders & Customer Tier     Shipment Tracking Logs
 (Help Center Articles)     (Relational Tables)        (Delivery Events)
```

To give an accurate "yes" or "no", the model needs three separate pieces of evidence:
1. **The Policy Article:** Opened electronics have a 14-day return window—unless the customer is on the *Plus* tier, which gives them 30 days.
2. **The Database Row:** Order `#ORD-402` is in the `Electronics` category, and this customer's tier is `Plus`.
3. **The Tracking Event:** The package for `#ORD-402` was delivered **19 days ago**.

When the model has all three facts, the logic is straightforward: 19 days is past the standard 14-day window, but within the 30-day *Plus* window, so the return is eligible.

### Why putting everything into one vector index falls short

If you serialize database rows and JSON events into text strings and store them in the same vector index as help center articles, three issues come up:

- **Embeddings are tuned for natural prose, not database rows.** Text embedding models are trained primarily on sentences and paragraphs. Comparing a conversational question against a serialized record like `order_id=ORD-402 | tier=Plus` often produces low cosine similarity.
- **Chunk sizes don't match.** A return policy needs 300–500 tokens of surrounding context to preserve its headings and exceptions. An order record is ~20 tokens. A single chunking strategy doesn't fit both well.
- **Long paragraphs crowd out short records.** When you retrieve the "Top 5" chunks globally, full paragraphs about return policies closely match the phrasing of the user's question and fill the top slots. The short database record (`tier=Plus`) easily falls below the top-$$K$$ cutoff. Without that record, the model only sees "19 days > 14 days" and reaches the wrong conclusion.

---

## 2. A Cleaner Pattern: Split the Search, Reserve the Slots

Rather than asking a single vector index to handle prose, relational rows, and event logs together, it helps to route each part of the query to the storage engine built for it—and reserve dedicated slots in the prompt for each source.

```text
                                 [User Query]
                                      │
                        [Query & Entity Breakdown]
                      │               │               │
       ┌──────────────┘               │               └──────────────┐
       ▼                              ▼                              ▼
[Source 1: Help Docs]          [Source 2: SQL DB]            [Source 3: JSON Logs]
 query: "opened box             lookup: order_id =            filter: order_id =
  electronics return policy"            ORD-402                       ORD-402
       │                              │                              │
 Vector + Keyword Search        Exact SQL Query               Timestamp Filter
       │                              │                              │
 Merge (RRF) + Re-Rank                │                              │
       │                              │                              │
  [Top 3 Policy Chunks]         [1 Order Record]              [1 Delivery Event]
       └──────────────┬───────────────┴───────────────┬──────────────┘
                      ▼                               ▼
                 [Combined Context Sent to LLM (5 Slots Total)]
```

### Step 1: Match the retrieval method to the data type

| Source | Format | Retrieval Approach | Reserved Prompt Slots |
| :--- | :--- | :--- | :--- |
| **Help Docs** | Markdown, PDFs, FAQs | Hybrid search (Vector + Keyword), merged with RRF and re-ranked. | **3 slots** |
| **Orders DB** | SQL tables (`orders`, `users`) | Extract `ORD-402` from the question and run a parameterized SQL lookup. | **1 slot** |
| **Tracking Logs** | JSON delivery events | Filter by `order_id = ORD-402` and select the latest delivery event. | **1 slot** |

### Step 2: Blend vector and keyword search inside the docs (RRF)

Even within unstructured help articles, pure vector search can miss exact terms. Vectors are good at matching paraphrases (*"opened the box"* $$\leftrightarrow$$ *"broken factory seal"*), while keyword search (BM25) is good at matching specific product categories or tier names (`Electronics`, `Plus`, `14-day`).

Because a cosine similarity score (`0.81`) and a BM25 score (`16.4`) sit on different scales, you can't simply add them together. A standard solution from information retrieval is **Reciprocal Rank Fusion (RRF)**, which ignores raw scores and combines the **rank** ($$r$$) each search method assigned to a document:

$$
\text{RRF Score} = \frac{1}{60 + r_{\text{vector}}} + \frac{1}{60 + r_{\text{keyword}}}
$$

If an article ranks near the top in both vector search and keyword search, its combined score rises naturally without manual weight tuning.

> **A useful boundary:** RRF works well *inside* a single document collection where both retrievers rank the same pool of articles. It is less helpful across separate data sources (like SQL tables vs. Markdown docs) whose IDs never overlap. Exact database lookups shouldn't have to compete with narrative paragraphs for a top-$$K$$ ranking slot.

### Step 3: Re-rank the text, keep exact records in their own slots

Once RRF returns the top 15 help article candidates, a Cross-Encoder re-ranker narrows those down to the 3 most relevant text chunks. The SQL order row and the JSON delivery event bypass the text re-ranker and go straight into their reserved slots—ensuring the model receives all three pieces of context.

---

## 3. Synthetic Benchmark Results

To test how much each step actually helps, I built a **60-question synthetic dataset** of multi-step retail support queries (returns, warranties, and shipping rules) paired with mock policy articles, SQL rows, and JSON delivery events. Every question required facts from all three sources to get the right answer.

| Configuration (60 Synthetic Queries) | All-3-Sources Recall | Policy Doc MRR | Exact Answer Accuracy | P95 Retrieval Time |
| :--- | ---: | ---: | ---: | ---: |
| **1. Single Vector Index (Baseline)** | 48.3% | 0.54 | 55.0% | **42 ms** |
| **2. Single Index + Hybrid RRF** | 61.7% | 0.72 | 66.7% | 49 ms |
| **3. Split Sources + Global Re-Ranker** | 73.3% | 0.84 | 76.7% | 112 ms |
| **4. Split Sources + RRF + Reserved Slots** | **93.3%** | **0.88** | **91.7%** | 108 ms |

A few takeaways from the experiment:

- **A single vector index missed over half the multi-source joins (`48.3%` recall).** It found the general return policy reliably, but frequently missed the short SQL row or tracking event. When a piece of context was missing, final answer accuracy dropped to `55.0%`.
- **Splitting sources worked best when paired with reserved slots (`73.3%` vs. `93.3%`).** In Configuration 3, SQL and JSON records were fetched separately but passed into a single global Cross-Encoder alongside the policy chunks. About a quarter of the time, longer policy paragraphs still pushed the short SQL record out of the top 5. Reserving dedicated slots for the SQL and JSON lookups improved full recall by **20 percentage points**.
- **Hybrid RRF improved policy ranking (`0.54` $$\rightarrow$$ `0.88` MRR).** Combining vector and keyword search inside the help articles helped surface specific policy exceptions (`Plus`, `Electronics`) near the top before re-ranking.

---

## 4. Practical Takeaways

1. **Keep deterministic lookups separate from semantic search.** When a query contains an explicit identifier like an Order ID or Ticket ID, fetch that record directly via SQL or an API and give it a dedicated slot in the context window. Save vector search and re-ranking for unstructured text.
2. **Keep section headings attached to text chunks.** Prefixing each policy chunk with its parent heading (`Returns > Opened Electronics`) prevents short exception rules from losing their context during retrieval.
3. **Test with "missing-data" queries too.** Including test cases with nonexistent order numbers (`#ORD-999`) or packages still in transit helps verify that the system reports what is missing rather than guessing a return window.
