# PageIndex

## 1. Overview
PageIndex is a **vectorless, reasoning-based RAG** system. It abandons traditional chunking and vector embeddings. Instead, it builds a hierarchical "Table of Contents" tree index from long documents and provides tools for an LLM agent to navigate this tree, effectively simulating how a human expert would skim a document's structure to find relevant sections before reading the full text.

## 2. Architecture Deep-Dive
- **Tree Construction (`page_index_md.py`, `page_index.py`)**: Parses the raw document (PDFs via OCR, or Markdown via heading levels) into a hierarchical tree of nodes. For each node/section, it uses an LLM to generate a concise summary.
- **Tools (`retrieve.py`)**: Exposes three key functions to the LLM agent:
  - `get_document()`: Gets document metadata.
  - `get_document_structure()`: Returns the tree structure (titles, node IDs, and summaries, but explicitly *without* the full text).
  - `get_page_content(pages)`: Fetches the raw text for specific pages/lines.
- **Retrieval Phase**: An agent (using standard frameworks like OpenAI Agents SDK) is given a query and these tools. It first calls `get_document_structure()` to see the Table of Contents + summaries. It reasons about which sections are most relevant, and then calls `get_page_content()` to read the raw text of those specific sections to answer the query.

## 3. Core Technical Approach
- **Strategy**: Semantic Tree Indexing + Agentic Navigation. 
- **Tradeoffs**: Because it relies entirely on the LLM to read the tree structure and decide what to fetch, it is highly explainable and avoids the "similarity ≠ relevance" trap of Vector DBs. The massive tradeoff is **indexing cost**: building the index requires calling the LLM to summarize *every single section* of the document.

## 4. What It's Used For
PageIndex is optimized for complex, long-form professional documents like financial reports, regulatory filings, and textbooks where multi-step reasoning is required and keyword/vector similarity fails. 

## 5. Relevance to Our Project
- **School of Thought**: PageIndex is a pure implementation of **B (Hierarchical bottom-up LLM summarization)** mixed with **C (Agentic translation)** at query time.
- **Transferability**: Very high conceptually. A codebase naturally forms a tree (Directories → Files → Classes → Functions). Building a "PageIndex" for a codebase means generating an LLM summary for every structural node and letting an agent navigate the tree. 
- **Concrete Mapping**: The `get_document_structure()` and `get_page_content()` tool pattern is the exact API we should expose to our mining agent. Provide the agent with the high-level codebase tree (with lightweight summaries) and let it fetch the raw code (`get_page_content`) only when it spots a potential AI/ML product opportunity in a module.
- **Addressing Specific Needs**: This strictly separates the "persistent knowledge representation" (the Tree of Summaries) from the "repeated mining" (the Agent traversing the tree).

## 6. Reusable Components / Patterns
- **Agentic Retrieval Workflow**: The separation of `get_document_structure()` (which strips out the heavy raw text to save context window) and `get_page_content()` is a brilliant and highly effective pattern for agentic codebase exploration. We should absolutely expose our codebase representation using this two-step tool pattern.

## 8. Limitations & Risks
- **High Indexing Cost**: Just like GraphRAG, generating an LLM summary for every node in the codebase tree is computationally expensive. PageIndex does not provide a mechanism to do this cheaply.
- **Not Built for Code**: It is designed for Markdown and PDFs. It splits Markdown by header tags (`#`, `##`). To use this pattern on code, we would have to replace its parser with a proper AST/Tree-sitter parser (like the one from `LocAgent`).

## 9. Verdict
**Medium-High** relevance. We won't use its parsing code, but its **Agentic Vectorless RAG architecture** (giving the LLM the tree structure first, then letting it fetch code) is the perfect interaction model for our opportunity mining agent. It validates that we don't necessarily need a Vector DB if we have a good hierarchical summary tree.
