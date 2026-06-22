# LocAgent

## 1. Overview
LocAgent is a framework designed to tackle **code localization**—identifying precisely where in a codebase changes need to be made based on an issue description. It does this by deterministically parsing the codebase into a directed heterogeneous graph, which an LLM agent then uses to navigate and search (multi-hop reasoning) to find the target code elements.

## 2. Architecture Deep-Dive
- **`dependency_graph/build_graph.py`**: The core graph constructor. It relies purely on static analysis (Python's `ast` module) to parse the codebase without any LLM intervention.
- **Node Types**: Extracts 4 node types: `directory`, `file`, `class`, and `function`.
- **Edge Types**: Extracts 4 edge types: `contains`, `inherits`, `invokes`, and `imports`.
- **Storage**: The constructed `networkx.MultiDiGraph` is serialized and saved as a `.pkl` file per repository.
- **Agent (`auto_search_main.py` & `plugins/location_tools`)**: An LLM agent is given function-calling capabilities to traverse this pre-computed graph structure. The agent starts from a search query and iteratively hops through the dependency graph (`imports`, `invokes`, etc.) to narrow down the localization.

## 3. Core Technical Approach
- **Strategy**: Deterministic static analysis for indexing + Agentic traversal for querying. 
- **Tradeoffs**: By avoiding LLM usage during the indexing phase, LocAgent completely eliminates the exorbitant costs and hallucination risks of LLM-based graph extraction (like those seen in LightRAG or GraphRAG). The graph is 100% accurate relative to the AST. The tradeoff is that the graph lacks semantic summaries (e.g., "what does this function do in plain English?") until the agent explicitly asks the LLM to read the function's source code at query time.
- **Language Lock-in**: The current implementation relies on Python's native `ast` parser, meaning it is tightly coupled to Python repositories. 

## 4. What It's Used For
LocAgent is explicitly built for **bug/issue localization** (e.g., SWE-Bench). It finds the specific files and functions that need editing to resolve a described issue. 

## 5. Relevance to Our Project
- **School of Thought**: LocAgent is a textbook implementation of **A (Deterministic structural graphs, zero LLM cost)** for the indexing phase, paired with **C (Knowledge-graph agentic translation)** at query time.
- **Transferability**: Very high for the *indexing methodology*. LocAgent proves that we do not need to spend thousands of dollars running LLMs over every file to extract entities and relationships. We can extract a highly accurate dependency graph deterministically. 
- **Concrete Mapping**: We should adopt LocAgent's ontology (`directory`, `file`, `class`, `function` nodes; `contains`, `inherits`, `invokes`, `imports` edges). However, instead of using Python's `ast`, we should use Tree-sitter to make the parser language-agnostic. 
- **Addressing Specific Needs**: LocAgent solves the problem of "how to build the graph cheaply and accurately". It lacks the persistent "semantic" layer we need for opportunity mining, but it provides the perfect structural foundation.

## 6. Reusable Components / Patterns
- **Ontology Design**: The mapping of code elements to graph nodes and edges in `build_graph.py` is an excellent, minimal, and sufficient design.
- **Separation of Concerns**: Building the graph entirely offline using static analysis, saving it, and giving an LLM agent tools to query it is a highly scalable pattern.

## 8. Limitations & Risks
- **Single Language Support**: Hardcoded to use `ast.parse`. To reuse this concept in our multi-module system, we'd need to rewrite the extraction logic using Tree-sitter.
- **Missing Semantic Context**: Because the graph only contains structural data (names and edges), identifying "AI/ML product opportunities" requires the LLM to fetch and read the raw code of the nodes at query time. There are no pre-computed text embeddings or semantic summaries attached to the nodes to enable fast semantic similarity search (unlike KGCompass or LightRAG).

## 9. Verdict
**High** relevance for structural indexing. LocAgent provides the ideal blueprint for our baseline codebase representation: a deterministic, zero-LLM-cost structural graph. To adapt this for "opportunity mining", we would combine LocAgent's deterministic structural extraction with LightRAG/KGCompass's semantic embedding layer.
