# Understand-Anything

## 1. Overview
Understand-Anything is a multi-agent system designed to turn a codebase into an interactive, visual knowledge graph. It provides a web dashboard for humans to explore the architecture, as well as an agentic chat interface to query the codebase.

## 2. Architecture Deep-Dive
- **Hybrid Extraction Pipeline**:
  - **Structural (Deterministic)**: Uses Tree-sitter to parse the code and extract absolute structural facts: imports, function/class definitions, call sites, inheritance. This generates the nodes and edges (e.g., `contains`, `imports`, `calls`).
  - **Semantic (LLM)**: Analyzes the parsed code to generate plain-English summaries (`fileSummary`, `functionSummaries`, `classSummaries`), tags, complexity scores, and architectural layer assignments (e.g., "API", "Data").
- **Incremental Processing**: To mitigate the high token costs of running LLMs over every file, it uses fingerprint-based change detection to only re-run the LLM analysis on files that have changed since the last commit.
- **Graph Visualization**: Outputs a massive `knowledge-graph.json` that powers a custom, force-directed UI dashboard.

## 3. Core Technical Approach
- **Strategy**: Deterministic structure + Agentic Semantic Summarization.
- **Tradeoffs**: It provides the "best of both worlds" (absolute structural accuracy from Tree-sitter + rich semantic context from LLMs). The tradeoff is the initial indexing cost, which is extremely high because it must summarize the entire codebase once. It mitigates this via incremental updates.

## 4. What It's Used For
Primarily used as an IDE plugin (Claude Code, Cursor, Copilot) to help human developers onboard onto new codebases. The developer opens a UI dashboard to click around the graph, take guided tours, and ask architectural questions.

## 5. Relevance to Our Project
- **School of Thought**: Understand-Anything is a Hybrid of **A (Deterministic structural graphs)** and **B (Hierarchical bottom-up LLM summarization)**.
- **Transferability**: Very high. This project perfectly validates our intended design: you *must* separate the deterministic structural parsing from the semantic LLM summarization. Tree-sitter guarantees that the dependency graph is 100% accurate, while the LLM layer provides the semantic context needed for opportunity mining. 
- **Concrete Mapping**: The pipeline defined here (`Tree-sitter parse -> LLM file summary -> merge into graph -> save to JSON`) is the exact pipeline we should build for our "once, queried/mined repeatedly" index.

## 6. Reusable Components / Patterns
- **Incremental LLM Analysis**: The concept of hashing files/ASTs to skip LLM summarization on unchanged files is mandatory for our system to be cost-effective.
- **Node Meta-data schema**: In `graph-builder.ts`, the metadata schema for nodes (`summary`, `tags`, `complexity`) is a great, minimal set of fields to ask the LLM to generate for each file/function.

## 8. Limitations & Risks
- **Overkill Visualization**: The project invests heavily in the UI layer (Dashboard, Guided Tours, visual clustering), which is irrelevant for our autonomous AI/ML opportunity mining agent.
- **Initial Cost**: The very first `/understand` run on a 200k LOC repo will still cost a massive amount of tokens. 

## 9. Verdict
**Very High** relevance for architectural validation. Understand-Anything perfectly proves that the optimal "Knowledge Representation" for a codebase is a deterministic AST-based graph *annotated* with LLM-generated semantic metadata. We should adopt this exact hybrid architecture for our indexing phase.
