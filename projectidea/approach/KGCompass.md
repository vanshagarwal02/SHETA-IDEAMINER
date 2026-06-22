# KGCompass

## 1. Overview
KGCompass is an evidence-grounded context-fusion system for repository-level bug localization and repair. It exists to solve the problem of accurately identifying the specific files and methods in a large codebase that need to be modified to fix a given bug (as described in an issue report), and preparing a fixed-budget context window to feed into an LLM for automated program repair. It was explicitly built and optimized for the SWE-bench benchmark.

## 2. Architecture Deep-Dive
- **`kgcompass/knowledge_graph.py`**: The core graph database manager. It connects to a Neo4j instance and handles the creation and updating of nodes (`Issue`, `Method`, `Class`, `File`, `Directory`, `Commit`, `Experience`) and relationships (`RELATED`).
- **`kgcompass/fl.py` (Fault Localization)**: Handles the ingestion of repository data and issue reports, coordinating the parsing of code, running git blame, and populating the Neo4j database with nodes and their relationships.
- **`kgcompass/language_factory.py`**: Provides language-agnostic parsing by delegating AST extraction (classes, methods, signatures) to specific parsers based on file extensions.
- **Data Flow**: An issue report is ingested and an issue-only LLM localizer proposes initial candidate files/methods. In parallel, the repo is parsed into a Neo4j KG (files, classes, methods, embeddings). The system then performs "file-local path mining", combining the LLM's predictions with the KG's structural and semantic connections to create a fused, top-k context window that is fed to the repair agent.
- **Key Design Patterns**: Graph Data Science (GDS) projection is used to create in-memory similarity graphs (`_create_similarity_projection` in `knowledge_graph.py`) to run graph algorithms over the code structure. 

## 3. Core Technical Approach
- **Strategy**: The central technique is fusing a repository-level Knowledge Graph (which captures structural AST relationships and semantic embeddings) with the output of an issue-only LLM localizer. It queries the graph to find nodes that are semantically similar to the issue and structurally close to other suspected nodes.
- **Tradeoffs**: It trades off the simplicity of a flat vector store for the precision of a structural graph database (Neo4j). It also optimizes heavily for *precision over a narrow scope* (bug localization) rather than *comprehensive summarization* (like HCAG). 
- **Implementation Details**: It uses Neo4j's GDS library to compute cosine similarity between node embeddings and text Levenshtein distance simultaneously directly in Cypher queries (e.g., `get_all_methods` in `knowledge_graph.py` combines `gds.similarity.cosine` and `apoc.text.levenshteinSimilarity`).

## 4. What It's Used For
It was actually built for **bug localization and automated program repair** on the SWE-bench benchmark. It explicitly processes "Bug report title/body" and outputs a "Repair agent / SWE-bench evaluator" context window.

## 5. Relevance to Our Project
- **School of Thought**: KGCompass belongs predominantly to **C (Knowledge-graph + agentic natural-language-to-query translation)**, though it relies more on programmed Cypher queries than agentic generation, and **A (Deterministic structural graphs)** as it builds a strict structural AST graph in Neo4j.
- **Transferability**: It was built to solve a completely different problem (bug localization). The fault localization (`fl.py`), issue-linking, and repair mechanisms DO NOT transfer to opportunity mining. However, the mechanism of persisting the codebase structure (files, classes, methods) and their semantic embeddings into a Neo4j graph (`knowledge_graph.py`) transfers cleanly to opportunity mining. We could use a similar graph to find all "Data" classes and query their relationships to "Service" classes to spot missing AI prediction features.
- **Concrete Mapping**: The `KnowledgeGraph` module in `kgcompass/knowledge_graph.py` does AST-to-Neo4j persistence → relevant to our persistence/cheap-rebuild question because Neo4j allows updating single node properties (`_create_and_link` handles updating an existing method's embedding and source code if it changes) without rebuilding the whole graph.
- **Addressing Specific Needs**: This helps with **persistence and cheap-rebuild-after-small-changes**. Updating the graph only requires re-parsing the changed files and executing `MERGE` or `SET` Cypher queries to update the affected method/class nodes, which is very cheap. It does *not* help with the opportunity mining process itself, as its retrieval logic is focused on finding the root cause of a specific bug.

## 6. Reusable Components / Patterns
- **`kgcompass/knowledge_graph.py`**: The `KnowledgeGraph._create_and_link` method is a great reference for how to efficiently index source code methods into Neo4j, avoiding index limits by separating the creation of searchable fields from large text properties.
- **`kgcompass/knowledge_graph.py`**: The Cypher query in `get_all_methods` demonstrates how to perform a hybrid search (semantic + lexical similarity) directly in Neo4j using the GDS and APOC libraries.

## 8. Limitations & Risks
- **Over-engineered for bugs**: The codebase is tightly coupled to SWE-bench dataset formats, GitHub Issue parsing (`fl.py`), and repair agents (`repair_claude.py`).
- **Heavy Infrastructure**: It requires running a Neo4j instance with GDS and APOC plugins, which adds significant operational overhead compared to a local SQLite database (like Aider uses).
- **Misleading Assumptions**: If copied uncritically, its assumption that the goal is to find the *single most relevant* localized subgraph (a "bug's search space") would actively mislead an opportunity mining system, which often needs to synthesize patterns across widely distributed components.

## 9. Verdict
**Medium** relevance. While the heavy coupling to bug localization makes the end-to-end pipeline unusable for opportunity mining, its `KnowledgeGraph` component provides a highly citable, robust pattern for persisting and incrementally updating repository ASTs and embeddings in Neo4j.
