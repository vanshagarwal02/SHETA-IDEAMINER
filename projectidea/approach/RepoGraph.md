# RepoGraph

## 1. Overview
RepoGraph provides a repository-level code graph to enhance LLM software engineering agents (specifically targeting SWE-bench issue resolution). It parses a codebase into a structural graph of definitions and references and provides a `search_repo()` tool for agents to query it.

## 2. Architecture Deep-Dive
- **Graph Construction (`construct_graph.py`, `utils.py`)**: 
  - Like LocAgent, RepoGraph uses deterministic static analysis (via Python's `ast` module and `tree-sitter`) to parse the codebase. 
  - It extracts classes, functions, and their hierarchical structures (`create_structure` in `utils.py`).
  - It uses tree-sitter queries to find definitions (`def`) and references/calls (`ref`).
  - It generates two artifacts: a `.jsonl` tags file (similar to `ctags`) containing line-level metadata for every entity, and a `.pkl` file containing the `networkx` graph.
- **Agent Integration**: 
  - Provides a `search_repo(search_term)` function to the LLM agent. 
  - When the agent searches for a class or function, it returns the definition and all reference relations from the pre-computed graph.

## 3. Core Technical Approach
- **Strategy**: Deterministic static analysis (AST/Tree-sitter) for indexing + Agentic traversal tool for querying.
- **Tradeoffs**: Same as LocAgent. Highly accurate, zero LLM cost during indexing, but lacks a pre-computed "semantic summary" layer. The agent has to query the graph structurally and fetch the raw code to understand semantics.

## 4. What It's Used For
Bug localization and automated code editing for SWE-bench (integrated into frameworks like Agentless and SWE-agent). It provides the agent with the repository context needed to fix bugs.

## 5. Relevance to Our Project
- **School of Thought**: Firmly belongs to **A (Deterministic structural graphs, zero LLM cost)**.
- **Transferability**: Validates the LocAgent pattern. It proves that extracting a structural graph using AST/Tree-sitter and exposing a graph-search API to an agent is a highly effective, low-cost way to make a codebase understandable to an LLM.
- **Concrete Mapping**: The `search_repo()` tool pattern (returning definition + references for a node) is exactly the kind of deterministic tool our opportunity mining agent will need to traverse the codebase graph. 

## 6. Reusable Components / Patterns
- **Tree-sitter Queries**: The `tree-sitter` queries in `construct_graph.py` (extracting `name.definition.class`, `name.definition.function`, `name.reference.call`) are a practical example of how to extract the relationships we need. We can adapt these tree-sitter queries for multiple languages.

## 8. Limitations & Risks
- **No Semantic Layer**: Relies purely on structural relationships (calls, defines). Without LLM summaries or embeddings attached to these nodes, "mining for AI product opportunities" (which requires understanding semantic intent) will be slow because the agent must fetch raw code constantly.

## 9. Verdict
**Medium** relevance. It is very similar to LocAgent. The primary value here is the validation of the deterministic Tree-sitter parsing approach and the specific `search_repo` API design. We will likely use the LocAgent/RepoGraph methodology for our base structural graph, and augment it with the semantic summarization layer of LightRAG/PageIndex.
