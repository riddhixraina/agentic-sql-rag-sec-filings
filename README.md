# Agentic SQL + RAG Analyst for SEC 10-K Filings

A Gemini agent that answers financial questions about 15 Form 10-K filings (3M, Adobe, Amazon, AMD and Apple, fiscal years 2020-2022). It decides per question whether to query a SQL database of exact figures, search the filing text, or do both, and it cites the document and printed page for every figure.

## Architecture

```
15 10-K PDFs ──PyMuPDF──▶ 1,897 pages (with printed page numbers)
      │
      ├─ ETL: statement parsing ─▶ DuckDB  financial_metrics  (120 validated rows, 8 metrics)
      │
      └─ chunking + all-MiniLM-L6-v2 ─▶ ChromaDB  (6,248 chunks, company/year metadata)

question ─▶ Gemini agent (tool calling)
              ├─ query_financial_database   read-only SQL: lookup, rank, growth, ratio
              └─ semantic_search            vector search, filtered by company and year
           ─▶ grounding check ─▶ answer + sources + tool trajectory
```

- **ETL.** Each filing orders its columns differently (Apple puts the newest year on the left, Amazon on the right), so the parser reads the year headers on each statement page and keeps the column for that filing's fiscal year. Label variants map to 8 canonical metrics (for example "Net sales" and "Total net sales" both map to Revenue, and Amazon's "Technology and content" maps to R&D).
- **Safety.** The database is opened read-only and only `SELECT` / `WITH` queries run.
- **Grounding.** A final answer that cites a number the tools never returned is rejected and regenerated from the tool output.

## Results (development set)

| Type | Question | Answer |
|---|---|---|
| SQL | Highest revenue in FY2022 | Amazon, $513,983M (AMAZON_2022_10K p.37) |
| SQL | Largest R&D growth 2020-2022 | AMD, +152.4% ($1,983M to $5,005M) |
| Semantic | AMD 2022 supply-chain risks | Third-party foundry dependence, limited control over yield and delivery |
| Semantic | Adobe 2022 competitive environment | Intense competition in Digital Experience, large established rivals |
| Hybrid | Largest revenue growth, and why | AMD +141.7%, Data Center +64% on EPYC, Gaming +21%, Xilinx (pp. 40, 45) |
| Hybrid | Highest R&D/revenue ratio, and focus | AMD 21.21%, with its stated R&D focus areas |

The notebook's failure analysis covers two bugs that were found and fixed: a parser that dropped amounts like `1,994` as if they were years, and retrieval that returned auditor "critical audit matters" chunks for a "what explains performance" question.

## Run it

```bash
pip install -r requirements.txt
export GEMINI_API_KEY=...        # or a key file / Colab secret, see the agent cell
jupyter notebook agentic_rag_sec_filings.ipynb
```

The 10-K PDFs download automatically from the public [FinanceBench](https://github.com/patronus-ai/financebench) dataset.

Built for NYU CS-GY 9223 Generative AI Engineering (Fall 2026).
