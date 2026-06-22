# RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval)

## 1. Overview
RAPTOR is a framework that improves RAG by building a recursive tree structure of summaries from text documents. It addresses the limitation of traditional vector RAG, which only retrieves small chunks of text and loses the global context. By recursively clustering and summarizing text, RAPTOR can answer queries that require high-level, holistic understanding of a document.

## 2. Architecture Deep-Dive
- **Chunking**: Splits the raw text into chunks.
- **Clustering**: Uses a clustering algorithm (e.g., Gaussian Mixture Models on embeddings) to group semantically similar chunks together.
- **Summarization**: Uses an LLM to generate a summary for each cluster.
- **Recursive Tree Building**: Treats the generated summaries as new text nodes and repeats the clustering + summarization process. This builds a hierarchical tree from the bottom up, where the root node is a summary of the entire document.
- **Retrieval**: When a query comes in, RAPTOR traverses the tree to retrieve nodes (which could be high-level summaries or low-level chunks) that are most semantically relevant to the question.

## 3. Core Technical Approach
- **Strategy**: Semantic Clustering + Recursive LLM Summarization.
- **Tradeoffs**: It provides excellent multi-hop reasoning capabilities by allowing the LLM to access high-level, synthesized context. The tradeoff is the high token cost of generating summaries for every cluster at every level of the tree.

## 4. What It's Used For
Sensemaking and question-answering over large, unstructured documents where queries often require synthesizing information scattered across the text.

## 5. Relevance to Our Project
- **School of Thought**: Belongs strictly to **B (Hierarchical bottom-up LLM summarization)**.
- **Transferability**: Very high conceptually, but the *clustering* implementation is unnecessary for us. Codebases already have an explicit, perfect hierarchical structure (Functions -> Classes -> Files -> Directories -> Modules). We do not need to compute embeddings and run Gaussian Mixture Models to figure out which pieces of code belong together.
- **Concrete Mapping**: We should adopt RAPTOR's *recursive summarization* pattern, but apply it to the deterministic AST graph (from LocAgent/RepoGraph) rather than semantic clusters. We can use an LLM to summarize files, and then recursively summarize directories based on their child file summaries, building up to a root summary of the entire codebase.

## 6. Reusable Components / Patterns
- **Recursive Summarization**: The idea of summarizing the summaries to build a hierarchical tree is essential for allowing our opportunity mining agent to quickly grasp the high-level architecture without reading thousands of low-level function summaries.

## 8. Limitations & Risks
- **Cost**: Like GraphRAG and PageIndex, generating all these summaries is expensive and requires incremental caching to be viable for codebases.
- **Semantic Clustering is Flawed for Code**: Using vector embeddings to cluster code chunks often groups syntactically similar code (e.g., all `catch` blocks) rather than functionally related code. Relying on the explicit directory structure is much safer.

## 9. Verdict
**Moderate** relevance. The recursive summarization concept is excellent and validates our need for a hierarchical semantic layer. However, its specific clustering methodology is designed for unstructured text and is not the right tool for structured codebases. We will achieve the "RAPTOR" tree effect simply by summarizing the deterministic AST graph.
