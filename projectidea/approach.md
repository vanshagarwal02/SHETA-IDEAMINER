# Repository Knowledge & Idea Mining Approaches

This document summarizes the core architecture, structure, and mechanics of the 19 approaches listed in the research dossier from `1.md`.

## 1. Aider — Repo Map
**Architecture / Mechanism:** Pure static analysis (zero LLM cost).
**How it works:** Uses `tree-sitter` to parse source files into an Abstract Syntax Tree (AST), extracting definitions and references. It builds a directed graph of files and symbols. A personalized PageRank algorithm is run over this graph to score importance, heavily weighting symbols referenced by currently open or edited files. Top-ranked symbols are selected via binary search to fit the LLM's token budget.

## 2. RepoGraph
**Architecture / Mechanism:** Line-level Code Graph with Subgraph Retrieval.
**How it works:** Represents the repository as a line-level graph capturing structural and relational information across files. Rather than summarizing the whole repo, it retrieves a specific "ego-network" (a subgraph of relevant lines and their immediate neighbors) to augment the LLM's context.

## 3. CodexGraph
**Architecture / Mechanism:** Graph Database + Natural Language to Cypher.
**How it works:** Indexes the repository into a Neo4j graph database using a unified schema for nodes (modules, classes, functions) and edges (contains, inherits, calls). An LLM agent translates natural language questions into Cypher queries to execute precise, multi-hop lookups against this graph database.

## 4. KGCompass
**Architecture / Mechanism:** Repository-Aware Knowledge Graph for Bug Repair.
**How it works:** Runs on Neo4j and links repository artifacts (like issues and PRs) directly to codebase entities. It uses "entity path tracing" to significantly narrow the search space of a bug down to a small set of highly relevant functions.

## 5. CGM — Code Graph Model
**Architecture / Mechanism:** Graph-Integrated LLM Attention.
**How it works:** Integrates the repository's code-graph structure directly into the LLM's attention mechanism. It uses a specialized adapter to map graph node attributes into the LLM's input space within an "agentless" graph-RAG framework.

## 6. HCAG
**Architecture / Mechanism:** Hierarchical Bottom-Up Summarization.
**How it works:** Generates recursive summaries from the bottom up. Leaf nodes (files) get detailed LLM summaries. Directory nodes aggregate child summaries into higher-level architectural overviews. A "compression-depth" parameter allows deeper directories to remain as placeholders until explicitly expanded on-demand.

## 7. Code-Craft / HCGS
**Architecture / Mechanism:** Multi-Level Graph with Granular Summarization.
**How it works:** Combines structural code graphs with LLM summarization at varying levels of granularity. This explicitly addresses the limitation of pure structural graphs, which often lack the ability to abstract details meaningfully at higher levels.

## 8. RAPTOR
**Architecture / Mechanism:** Recursive Abstractive Processing (Tree-Organized Retrieval).
**How it works:** A general-text RAG pattern. It clusters and summarizes text chunks recursively from the bottom up into a multi-layer tree. At query time, retrieval can seamlessly pull from any layer, enabling both granular detail and global overview lookups.

## 9. GraphRAG (Microsoft) & LightRAG
**Architecture / Mechanism:** LLM-Extracted Entity Graph + Community Detection.
**How it works:** Extracts an entity-relationship graph via an LLM. It then runs hierarchical community detection (Leiden algorithm) to cluster the graph at multiple levels. It generates bottom-up LLM summaries for each community. Queries use a map-reduce process across these community summaries to answer global corpus questions.

## 10. RepoUnderstander / LingmaAgent
**Architecture / Mechanism:** Lightweight Knowledge Graph + Search-Guided Agent.
**How it works:** Condenses the repo into a lightweight knowledge graph. Instead of pre-summarizing everything, it uses Monte Carlo Tree Search (MCTS) to guide an agent's active exploration of the graph, planning where to look on-demand.

## 11. LocAgent
**Architecture / Mechanism:** Directed Heterogeneous Graphs + Multi-Hop Agent Search.
**How it works:** Parses the codebase into a lightweight, heterogeneous graph (files, classes, functions, invokes). LLM agents perform multi-hop search and bug localization directly over this graph structure, significantly reducing LLM costs.

## 12. RANGER
**Architecture / Mechanism:** Two-Path Querying with MCTS Graph Exploration.
**How it works:** Uses `tree-sitter` to build language-agnostic file-level graphs, resolving intra-file structure before stitching them into a unified repo-level graph. Simple queries are answered directly, while complex queries trigger an MCTS-based exploration of the graph.

## 13. Understand-Anything
**Architecture / Mechanism:** Multi-Agent Pipeline to JSON Knowledge Graph.
**How it works:** A mature pipeline of specialized agents (scanner, analyzer, reviewer, tour-builder) that process files and output a single JSON knowledge graph. It is designed to be committed to version control to save computation costs, using post-commit hooks for incremental diff-impact analysis.

## 14. Code-Graph-RAG
**Architecture / Mechanism:** In-Memory Graph DB (Memgraph) + MCP Server.
**How it works:** Uses `tree-sitter` to parse multi-language monorepos into a unified graph schema stored in Memgraph. It exposes this database as an MCP server, allowing LLM clients to query via Cypher, perform semantic search, and execute AST-targeted code edits.

## 15. Knowledge Graph Based Repo-Level Code Gen
**Architecture / Mechanism:** Hybrid Retrieval for Code Generation.
**How it works:** Builds a repository knowledge graph and uses hybrid retrieval (graph + semantic) to extract a relevant subgraph. This subgraph is fed to an LLM to generate code that strictly adheres to the inter-file dependencies and modular conventions of the repository.

## 16. Awesome-Repo-Level-Code-Generation
**Architecture / Mechanism:** Curated Bibliography.
**How it works:** A continuously updated collection of papers and tools related to repository-level code generation and issue resolution.

## 17. Legacy Code Modernization with LLMs
**Architecture / Mechanism:** Living Documentation Generation.
**How it works:** Focuses on generating documentation to understand old/legacy code so it can be safely modified, rather than directly mining for new product opportunities.

## 18. Commercial GenAI Use Case Discovery Tooling
**Architecture / Mechanism:** Business Process & Operational Data Mining.
**How it works:** Analyzes business logs, tickets, and workflows to rank AI automation opportunities by ROI. Notably, these do *not* perform static analysis on the source code itself, establishing a clear gap in the market for code-level opportunity mining.

## 19. PageIndex
**Architecture / Mechanism:** Vectorless, Hierarchical Tree Search.
**How it works:** Originally designed for long PDFs. Builds a hierarchical table of contents tree with nodes containing summaries and exact pointers. Querying involves reasoning-based tree search instead of vector similarity, tracing back to the exact relevant section.
