# GraphRAG

## 1. Overview
GraphRAG is Microsoft Research's implementation of Graph Retrieval-Augmented Generation. It is a data pipeline designed to extract a structured knowledge graph from unstructured text corpora using LLMs, cluster that graph into hierarchical communities, and use those community summaries to answer complex global and local user queries.

## 2. Architecture Deep-Dive
- **Entity & Relationship Extraction**: Unlike deterministic code parsers, GraphRAG chunks raw text and feeds it to an LLM with a prompt asking it to extract all entities (people, places, concepts) and their relationships.
- **Community Detection**: Once the graph is built, it uses community detection algorithms (like Leiden) to group densely connected entities into hierarchical clusters.
- **Hierarchical Summarization**: It uses the LLM to generate a plain-English summary for *every community* at every level of the hierarchy.
- **Retrieval**: 
  - *Local Search*: Given a query, it finds relevant entities (via vector embeddings) and retrieves their immediate subgraph (neighbors and relationships) to generate an answer.
  - *Global Search*: Given a broad query ("What is the main theme of the dataset?"), it maps the query across all high-level community summaries in parallel, then aggregates the results into a final answer.

## 3. Core Technical Approach
- **Strategy**: Pure LLM extraction + Hierarchical Summarization.
- **Tradeoffs**: It provides unparalleled capability to answer holistic, global questions about a dataset. However, the cost of the indexing phase is astronomically high. Every text chunk requires an LLM call to extract entities, and every resulting community requires an LLM call to summarize.

## 4. What It's Used For
Sensemaking over large, unstructured narrative text corpora (e.g., investigative journalism, large document dumps) where traditional vector RAG fails to capture the "big picture".

## 5. Relevance to Our Project
- **School of Thought**: The canonical implementation of **B (Hierarchical bottom-up LLM summarization)**.
- **Transferability**: As an indexing engine for a *codebase*, GraphRAG's approach is highly inefficient. Code already has an explicit, deterministic graph structure (AST, imports, calls). Using an LLM to "discover" relationships that a parser can extract for free is a massive waste of compute and risks hallucinations.
- **Concrete Mapping**: However, its *community summarization* and *Global Search* concepts are extremely valuable. Once we extract a deterministic graph (using methods from LocAgent), we can cluster the code (e.g., grouping by directories or namespaces to form "communities") and then use GraphRAG's summarization pattern. This allows the agent to answer "What is the overall architecture?" by reading the high-level community summaries.

## 6. Reusable Components / Patterns
- **Community Summaries for Global Queries**: The pattern of maintaining hierarchical summaries so an agent doesn't have to read all 10,000 files to understand the big picture is essential for our system.

## 8. Limitations & Risks
- **Cost**: The LLM-based extraction phase is too expensive and slow to run repeatedly on large codebases. 
- **Non-Deterministic**: Because it relies on LLMs for extraction, two runs on the same codebase might yield slightly different graphs. For code, we want structural guarantees.

## 9. Verdict
**Moderate** relevance. The *extraction* phase of GraphRAG is the wrong tool for codebases. But the *aggregation* phase (clustering nodes into communities and summarizing them) is the perfect solution for creating the hierarchical map that our opportunity mining agent will traverse. We should pair LocAgent-style deterministic extraction with GraphRAG-style community summarization.
