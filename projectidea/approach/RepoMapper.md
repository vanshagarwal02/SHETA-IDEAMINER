# RepoMapper

## 1. Overview
RepoMapper is a standalone CLI tool and MCP server that generates a highly optimized, token-bounded "map" of a software repository for LLMs. It is derived from the core "RepoMap" feature in Aider. Its primary goal is to provide an LLM with the most important structural context of a codebase without blowing up the context window.

## 2. Architecture Deep-Dive
- **Parsing**: Uses `tree-sitter` (via `grep-ast`) to parse source code into an AST and extract definitions and references across multiple languages.
- **Graph Construction**: Builds a structural dependency graph where nodes are files/symbols and edges are references/calls.
- **Importance Ranking**: Runs the **PageRank** algorithm over this dependency graph to score every file and symbol. Symbols that are heavily referenced globally get higher scores.
- **Token Optimization**: Given a maximum token limit (e.g., 2048 tokens), RepoMapper uses binary search to select the highest-scoring (via PageRank) nodes and edges that fit perfectly within that limit.
- **Output**: Generates a textual map showing the directory structure, file names, and the signatures of the most important classes/functions within those files.

## 3. Core Technical Approach
- **Strategy**: Deterministic Structural Graph + PageRank Compression.
- **Tradeoffs**: It completely abandons semantic summarization in favor of pure structural importance. It solves the LLM context window problem deterministically using graph algorithms (PageRank) rather than LLM summarization (GraphRAG/PageIndex).

## 4. What It's Used For
RepoMapper is used to give coding assistants (like Aider or Cline) a persistent, lightweight "awareness" of the entire repository so they know what files exist and what the core APIs look like, without actually reading every file.

## 5. Relevance to Our Project
- **School of Thought**: Belongs strictly to **A (Deterministic structural graphs, zero LLM cost)**.
- **Transferability**: Extremely high. The use of **PageRank** on a codebase AST graph is a brilliant mechanism for filtering. In our "Opportunity Miner" agent, we can't always feed the entire tree (like PageIndex does) if the codebase has 10,000 files. We can use RepoMapper's PageRank approach to condense the deterministic graph (built via LocAgent/RepoGraph methods) into a highly relevant subset to seed the agent's context.
- **Concrete Mapping**: 
  1. We build the deterministic graph (LocAgent).
  2. We run PageRank on the graph (RepoMapper) to find the "core" architecture.
  3. We feed this compressed map to the agent to start its opportunity mining.

## 6. Reusable Components / Patterns
- **PageRank for Context Selection**: The implementation of networkx PageRank in `repomap_class.py` to rank code elements is highly reusable.
- **Token-Aware Binary Search**: The logic to iteratively expand/contract the AST graph to perfectly fit a target token limit is a great pattern for robust LLM prompts.

## 8. Limitations & Risks
- **No Semantics**: Just like Aider, it only shows function signatures (`def foo(x):`). It strips out docstrings and implementation. The opportunity mining agent will still need a tool to fetch the raw implementation (like `get_page_content` from PageIndex) once it spots an interesting function signature in the map.

## 9. Verdict
**High** relevance. RepoMapper provides the exact filtering algorithm (PageRank) needed to make deterministic codebase graphs usable by LLMs at scale. We should incorporate its PageRank + token-bounding logic into our query interface to ensure our agent is never overwhelmed by large codebases.
