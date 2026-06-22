# IdeaMiner: Persistent Codebase Knowledge Representation for AI/ML Opportunity Mining

## The Problem, Stated Precisely

We want to turn a large, multi-module code repository into a **persistent, reusable knowledge representation** that can be **built once** and **queried/mined repeatedly** to surface AI/ML product-opportunity ideas. The existing approach — chunking raw source files and calling an LLM per chunk — works but is expensive, non-reusable (you re-read everything each time), and loses inter-module context (a chunk from `OrderService.java` doesn't know that `FraudDetectionService.java` exists three modules away and is calling the same data).

Every prior system we researched (Aider, LocAgent, RepoGraph, CodexGraph, KGCompass, GraphRAG, LightRAG, RAPTOR, PageIndex, Understand-Anything, HCAG, RANGER, Code-Graph-RAG) solves a *different* downstream task — code generation, bug localization, automated repair, codebase Q&A. None of them do opportunity mining. The knowledge representation they build is optimized for their task, not ours. We need to reason about which mechanisms transfer and which don't.

---

## Part 1: The Design

### What the knowledge representation must support

Our downstream consumer is an **opportunity-mining agent** that needs to:

1. **Understand the architecture** — "This is a Spring Boot monolith with 14 modules covering payments, auth, notifications, catalog, etc." — without reading 500,000 lines of code.
2. **Understand what each module does semantically** — "The `catalog` module manages product listings, categories, and search. It uses Elasticsearch and exposes REST endpoints consumed by the frontend and the recommendation engine."
3. **Understand data flow and integration points** — "The `order` module calls `inventory`, `payment`, and `notification` modules. Customer data flows from `user-profile` through `order` to `fraud-detection`."
4. **Identify patterns that indicate AI/ML opportunities** — rule engines that could become ML models, hardcoded thresholds that could be learned, text processing that could use NLP, manual classification that could be automated, customer-facing flows that could be personalized.
5. **Dig deeper on demand** — when a high-level scan flags "the pricing module has hardcoded discount rules," the agent needs to drill into the actual code of `PricingEngine.java` and `DiscountStrategy.java` to propose a specific ML replacement.

These requirements dictate the representation. Let me evaluate each school of thought against them.

---

### Weighing the Schools of Thought

#### School A: Deterministic Structural Graphs (Aider/RepoMapper, LocAgent, RepoGraph)

**What they actually do (from the code):**

Aider's `RepoMap` (in [repomap.py](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/aider/aider/repomap.py)) uses tree-sitter to extract `Tag(rel_fname, fname, line, name, kind="def"|"ref")` tuples, builds a `networkx.MultiDiGraph` where nodes are files and edges are cross-file symbol references (weighted by reference count), runs `nx.pagerank()` with personalization (50× boost for active files), and binary-searches for the maximum number of top-ranked definitions that fit a token budget. The output is a compact text string of file paths + function signatures.

LocAgent's `build_graph()` (in [build_graph.py](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/LocAgent/dependency_graph/build_graph.py)) uses Python's `ast` module to extract 4 node types (`directory`, `file`, `class`, `function`) and 4 edge types (`contains`, `imports`, `invokes`, `inherits`). Node IDs are `filename:ClassName.method_name`. The full source code is stored as a node attribute on file nodes. The graph is serialized as pickle.

RepoGraph's `CodeGraph` (in [construct_graph.py](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/RepoGraph/repograph/construct_graph.py)) uses tree-sitter queries to extract definitions and references, filters out stdlib/builtins, and also uses `ast.parse()` for structural analysis. It's Python-only despite using tree-sitter.

**What transfers to our problem:**
- ✅ The structural graph is the **foundation**. We absolutely need the `directory → file → class → function` hierarchy with `imports/calls/inherits` edges. This is free to compute, 100% accurate, and deterministic.
- ✅ Aider's **PageRank** is directly useful — not for "which file is the user editing" (we have no user), but for "which files are architecturally central." A file imported by 40 other files is more important than a utility imported by 2. This helps the mining agent prioritize where to look.
- ✅ Aider's **token budgeting** (binary search over ranked definitions) is essential for fitting codebase context into mining prompts.
- ✅ LocAgent's **containment hierarchy** (directory → file → class → function) is the correct ontology for code. RepoGraph's flat node list (no directory/file hierarchy) is worse for our purposes.

**What doesn't transfer:**
- ❌ The structural graph alone has **no semantic content**. It tells you that `PricingEngine.applyDiscount()` calls `RuleEngine.evaluate()`, but not *what* these functions do or *why* that call pattern is an ML opportunity. The mining agent would have to fetch and read the raw code of every function it encounters — which is what we're trying to avoid.
- ❌ LocAgent's Python-`ast`-only parser doesn't work for Java/Kotlin/TypeScript repos. We need tree-sitter (like Aider and RepoGraph use).

**Verdict:** Necessary but not sufficient. We use this as Layer 1.

---

#### School B: Hierarchical Bottom-Up LLM Summarization (HCAG, RAPTOR, GraphRAG community summaries)

**What they actually do:**

HCAG (paper only, no code) recursively summarizes a code repo bottom-up: leaf files get LLM summaries (key functionality, core logic, inputs/outputs, dependencies), then directory nodes aggregate their children's summaries into higher-level descriptions. It has a **compression-depth parameter C** — directories deeper than C get a placeholder instead of a full summary, expanded on-demand later.

RAPTOR ([raptor/](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/raptor/)) clusters text chunks by embedding similarity, summarizes each cluster, then recursively clusters the summaries. This makes no sense for code — code already has an explicit hierarchy (directories/packages).

GraphRAG ([graphrag/](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/graphrag/)) uses Leiden community detection to cluster an entity graph, then generates LLM reports per community. The community reports contain a title, summary, importance rating, and key findings.

PageIndex ([pageindex/](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/PageIndex/)) builds a tree-of-contents from documents. At query time, the agent first sees the tree structure (titles + summaries, **without** the raw text), reasons about which sections are relevant, then fetches raw content via `get_page_content()`. This two-step pattern (structure overview → selective drill-down) is precisely what we need.

**What transfers:**
- ✅ **File-level LLM summaries** are essential. The mining agent needs to know "this file implements a rule-based fraud scoring engine" without reading 800 lines of Java.
- ✅ **Recursive directory summaries** are essential. The mining agent needs to know "the `fraud-detection` module combines rule-based scoring with velocity checks and manual review queues" without reading 15 individual file summaries.
- ✅ HCAG's **compression-depth parameter** is a great idea — we don't need to pre-summarize every leaf utility file. We summarize the top N levels of the tree and expand deeper nodes on demand.
- ✅ PageIndex's **two-step retrieval pattern** (`get_structure()` then `get_content()`) is the correct API for our mining agent.

**What doesn't transfer:**
- ❌ RAPTOR's **embedding-based clustering** is unnecessary — code already has packages/modules/directories.
- ❌ GraphRAG's **Leiden community detection** is solving a problem we don't have (finding clusters in an unstructured entity graph). Our "communities" are just directories/packages.
- ❌ GraphRAG's **map-reduce global search** is designed for "answer a question across the entire corpus." We need something more targeted — "scan this module for opportunities."

**Verdict:** Essential for the semantic layer. We adopt file + recursive directory summaries, with a depth cutoff. We skip clustering algorithms entirely.

---

#### School C: Knowledge Graph + Agentic Cypher/Traversal (CodexGraph, Code-Graph-RAG, LocAgent's agent)

**What they actually do:**

CodexGraph stores the repo in Neo4j with a fixed schema (MODULE, CLASS, FUNCTION, METHOD nodes; CONTAINS, CALLS, INHERITS edges) and has the LLM generate Cypher queries. Code-Graph-RAG does the same with Memgraph.

LocAgent gives the agent tools (`get_neighbors()`, `view_code()`, `get_node_info()`) and lets it do multi-hop traversal through the graph.

**What transfers:**
- ✅ The **tool-based traversal** pattern (LocAgent) is better than Cypher generation for our use case. Our mining agent isn't asking precise structural queries ("find all classes inheriting from X") — it's exploring open-endedly ("what does this module do, and could any of it benefit from ML?"). Tools like `get_module_summary()`, `list_functions(file)`, `get_function_code(file, func)` are more natural.
- ❌ A **graph database** (Neo4j/Memgraph) adds operational complexity (Docker, ports, connection management) for marginal benefit. Our graph fits in memory (even a 500k-LOC repo produces a graph with ~10k nodes). NetworkX or a JSON file is sufficient.

**Verdict:** We adopt the tool-based traversal pattern, but store the graph as a JSON file, not a graph database.

---

#### School D: MCTS-Guided On-Demand Exploration (RepoUnderstander, RANGER)

**What they propose (papers only, no code):**

Build a lightweight graph, then use Monte Carlo Tree Search to guide an agent's exploration — the agent *plans* which parts of the repo to examine before acting. RANGER adds a two-path design: simple queries resolved directly, complex ones trigger MCTS.

**What transfers:**
- ✅ The **two-path insight** is useful: a broad "scan everything" pass should use the pre-computed summaries, while a targeted "dig into this module" pass should do on-demand exploration using the structural graph + raw code.
- ❌ Full **MCTS** is overkill for opportunity mining. MCTS optimizes for finding a single best path through a search space (like bug localization). Opportunity mining needs to *cover* the codebase broadly, not find one specific thing. A simpler strategy — iterate through modules by PageRank importance — works better.

**Verdict:** We adopt the two-mode insight (broad scan vs. targeted dig), but use simple iteration instead of MCTS.

---

### The Recommended Architecture

Based on the above analysis, the system has three layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 3: MINING AGENT                            │
│  Two modes: (a) Broad Scan  (b) Targeted Deep-Dive                │
│  Consumes the knowledge base via tool APIs                         │
│  Produces structured opportunity reports                           │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Tools: get_tree(), get_summary(),
                         │        get_code(), search_symbols()
┌────────────────────────▼────────────────────────────────────────────┐
│                LAYER 2: SEMANTIC ANNOTATION                         │
│  LLM-generated summaries attached to structural nodes              │
│  File summaries, function summaries, directory roll-ups            │
│  Incremental: only re-summarize structurally changed files         │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Reads from / writes to
┌────────────────────────▼────────────────────────────────────────────┐
│              LAYER 1: DETERMINISTIC STRUCTURAL GRAPH                │
│  Tree-sitter parsing → nodes (dir/file/class/function)             │
│  Edges (contains/imports/calls/inherits)                           │
│  PageRank scores, source code stored per-file                      │
│  Zero LLM cost, fully deterministic                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Persistence and Cost

### Storage Format

A single directory `.ideaminer/` at the repo root, containing:

```
.ideaminer/
├── manifest.json            # Version, repo name, last-analyzed commit hash
├── structural_graph.json    # The full Layer 1 graph (nodes + edges + PageRank scores)
├── summaries/
│   ├── file_summaries.json  # {file_path: {summary, tags, complexity, function_summaries}}
│   └── dir_summaries.json   # {dir_path: {summary, child_count, key_components}}
├── fingerprints.json        # {file_path: {content_hash, structural_hash, mtime}}
└── opportunities/           # Output of mining runs
    ├── scan_2026-06-22.json # Full scan results
    └── deep_payments.json   # Targeted deep-dive results
```

**Why JSON, not SQLite/Neo4j/pickle:**
- **JSON is human-readable** — developers can inspect and debug the knowledge base. This matters for a system that produces opportunity reports that will be reviewed by humans.
- **JSON is git-committable** — following Understand-Anything's pattern, the knowledge base can be committed to the repo so teammates skip re-running the pipeline.
- **JSON is sufficient** — even a 500k-LOC repo produces ~10k structural nodes. This is trivially small for in-memory processing. We load the graph into a NetworkX `MultiDiGraph` at runtime for traversal/PageRank.
- **Pickle is fragile** (LocAgent's approach) — pickle files break across Python versions and can't be inspected. JSON with a defined schema is better.

### The Structural Graph Schema (`structural_graph.json`)

Drawing directly from LocAgent's ontology (4 node types, 4 edge types) and Understand-Anything's schema (typed, validated):

```json
{
  "version": "1.0.0",
  "project": {
    "name": "my-ecommerce-app",
    "languages": ["java", "kotlin"],
    "root_hash": "abc123",
    "analyzed_at": "2026-06-22T10:00:00Z"
  },
  "nodes": [
    {
      "id": "dir:src/main/java/com/app/payments",
      "type": "directory",
      "name": "payments",
      "path": "src/main/java/com/app/payments",
      "pagerank": 0.0342
    },
    {
      "id": "file:src/main/java/com/app/payments/PricingEngine.java",
      "type": "file",
      "name": "PricingEngine.java",
      "path": "src/main/java/com/app/payments/PricingEngine.java",
      "language": "java",
      "loc": 847,
      "pagerank": 0.0089
    },
    {
      "id": "class:src/main/java/com/app/payments/PricingEngine.java:PricingEngine",
      "type": "class",
      "name": "PricingEngine",
      "path": "src/main/java/com/app/payments/PricingEngine.java",
      "line_range": [15, 847],
      "pagerank": 0.0067
    },
    {
      "id": "func:src/main/java/com/app/payments/PricingEngine.java:PricingEngine.applyDiscount",
      "type": "function",
      "name": "applyDiscount",
      "path": "src/main/java/com/app/payments/PricingEngine.java",
      "line_range": [120, 195],
      "pagerank": 0.0043
    }
  ],
  "edges": [
    {"source": "dir:...", "target": "file:...", "type": "contains"},
    {"source": "file:...", "target": "class:...", "type": "contains"},
    {"source": "class:...", "target": "func:...", "type": "contains"},
    {"source": "file:payments/PricingEngine.java", "target": "file:rules/RuleEngine.java", "type": "imports"},
    {"source": "func:...PricingEngine.applyDiscount", "target": "func:...RuleEngine.evaluate", "type": "calls"},
    {"source": "class:...PricingEngine", "target": "class:...AbstractPricer", "type": "inherits"}
  ]
}
```

### The Summary Schema (`file_summaries.json`)

Drawing from Understand-Anything's `LLMFileAnalysis` interface:

```json
{
  "src/main/java/com/app/payments/PricingEngine.java": {
    "summary": "Implements product pricing with configurable discount rules. Uses a rule engine to evaluate discount eligibility based on customer tier, cart value, and promotional campaigns. Supports percentage-based, fixed-amount, and tiered discounts.",
    "tags": ["pricing", "rules-engine", "business-logic", "discounts"],
    "complexity": "complex",
    "function_summaries": {
      "applyDiscount": "Evaluates all active discount rules against the current cart and applies the best matching discount. Falls back to a hardcoded 5% loyalty discount if no rules match.",
      "calculateTax": "Computes sales tax based on shipping state. Uses a lookup table of state tax rates.",
      "buildPriceBreakdown": "Constructs a detailed price breakdown object showing subtotal, discounts, tax, shipping, and total."
    },
    "ai_ml_signals": ["hardcoded discount rules", "rule engine pattern", "customer tier classification"],
    "structural_hash": "sha256:a1b2c3...",
    "summarized_at": "2026-06-22T10:05:00Z"
  }
}
```

> [!IMPORTANT]
> The `ai_ml_signals` field is new — not present in any system we studied. We ask the LLM to flag potential AI/ML-relevant patterns *during summarization*, not just during mining. This means the expensive per-file LLM call does double duty: it produces a reusable summary AND pre-flags signals that the mining pass can later aggregate.

### Incremental Update Strategy

Drawing from Understand-Anything's fingerprinting:

1. On each run, compute `(content_hash, structural_hash)` per file (Understand-Anything's [fingerprint.ts](file:///Users/vanshagarwal/Desktop/SHETA/ideaminer/projectidea/repos/Understand-Anything/understand-anything-plugin/packages/core/src/fingerprint.ts) approach).
2. Compare against stored fingerprints:
   - **No change** (`content_hash` matches): skip entirely.
   - **Cosmetic change** (`content_hash` differs, `structural_hash` matches — e.g., only comments/formatting changed): update the stored code but don't re-run LLM summarization.
   - **Structural change** (`structural_hash` differs): re-parse with tree-sitter, re-run LLM summarization, update graph.
3. After re-summarizing changed files, **re-roll-up directory summaries** only for directories that contain changed files.
4. Re-run PageRank on the updated structural graph (this is fast — milliseconds for 10k nodes).

**Cost model:** For a 500-file repo with 20 files changed in a typical PR:
- Layer 1 (structural): Re-parse 20 files with tree-sitter. Cost: ~0 (pure computation, sub-second).
- Layer 2 (semantic): Re-summarize 20 files with LLM. Cost: ~20 × $0.01 = $0.20 (assuming GPT-4o-mini, ~1k tokens input + ~200 tokens output per file).
- Directory roll-ups: Re-summarize ~5 ancestor directories. Cost: ~$0.05.
- Total incremental cost: **~$0.25** per update.

Compare to the existing chunked approach (re-reading the entire repo): 500 files × $0.01 = **$5.00** per run.

---

## Part 3: How Opportunity Mining Works

### The File Summarization Prompt (Layer 2)

This prompt runs once per file during indexing. It's the most important prompt in the system because it produces the reusable knowledge:

```
You are a senior software architect analyzing source code for a knowledge base.

## Context
Project: {{project_name}}
Module: {{module_path}} — {{module_summary_if_available}}
This file imports from: {{import_list}}
This file is imported by: {{reverse_import_list}}

## File
Path: {{file_path}}
Language: {{language}}

```{{language}}
{{file_content}}
```

## Task
Produce a JSON analysis of this file:

{
  "summary": "2-3 sentence description of what this file does and why it exists in the system.",
  "tags": ["up to 5 semantic tags"],
  "complexity": "simple | moderate | complex",
  "function_summaries": {
    "functionName": "1-sentence summary of what this function does"
  },
  "ai_ml_signals": [
    "List any patterns you observe that could indicate an AI/ML opportunity. Examples:",
    "- Hardcoded rules/thresholds that could be learned from data",
    "- Manual classification or categorization logic",
    "- Text processing that could use NLP models",
    "- Recommendation logic based on simple heuristics",
    "- Fraud/risk scoring using rule engines",
    "- Search relevance based on keyword matching",
    "- Content generation using templates",
    "- Data validation that could use anomaly detection",
    "If none are apparent, return an empty array."
  ]
}
```

> [!NOTE]
> Notice we provide **import context** (what this file imports and what imports it). This is drawn from the structural graph (Layer 1) and gives the LLM crucial inter-file context that raw chunking misses. This is the key advantage of having a structural graph before summarization.

### The Mining Agent (Layer 3)

The mining agent operates in two modes:

#### Mode A: Broad Scan

This is the "survey everything" mode. It reads the pre-computed knowledge base and identifies opportunities across the entire codebase. It does NOT re-read raw source code.

**Input to the agent:** The full directory tree with summaries, plus all pre-flagged `ai_ml_signals` aggregated by module.

**Process:**
1. Load `dir_summaries.json` → present the top-level project architecture to the LLM.
2. For each top-level module (sorted by PageRank importance):
   - Present the module's directory summary + the `ai_ml_signals` from all files in that module.
   - Ask the LLM to evaluate each signal: Is this a real opportunity? What category does it fall into? What's the business value? What would the ML solution look like?
3. Cross-reference across modules: "The fraud-detection module's rule engine and the pricing module's discount rules both use the same `RuleEngine` class — this is a broader pattern of rule-based decision-making that could be replaced with ML models."

**Prompt (per module):**
```
You are an AI/ML product strategist analyzing a software module for automation and intelligence opportunities.

## Module: {{module_name}}
{{module_summary}}

## Files in this module (with AI/ML signals):
{{#each files}}
### {{file.name}} ({{file.complexity}})
{{file.summary}}
Signals: {{file.ai_ml_signals}}
{{/each}}

## Cross-module context
This module is imported by: {{importing_modules}}
This module imports: {{imported_modules}}

## Task
For each genuine AI/ML opportunity you identify:
1. Category (from: Personalization, Proactive Intelligence, NL Interfaces, Smart Automation, Risk/Fraud, Dynamic Content & GenAI, Accessibility, Model Replacement of Rules, Observability/Anomaly Detection, Domain Predictors, Information Extraction)
2. Specific opportunity description
3. Current implementation (cite the file and function)
4. Proposed ML/AI approach
5. Data requirements
6. Estimated complexity (low/medium/high)
7. Business value rationale

Return as a JSON array of opportunities.
```

**Cost:** For a 14-module project, this is ~14 LLM calls reading ~2k tokens each (summaries, not raw code). Total: **~$0.15** per full scan.

#### Mode B: Targeted Deep-Dive

This is the "investigate this specific area" mode. Triggered when:
- The broad scan flags an interesting signal and the user (or agent) wants to go deeper.
- A user explicitly asks: "Look for ML opportunities in the payments module."

**Process:**
1. Load the structural graph for the target module.
2. For each file in the module (sorted by PageRank), fetch the **raw source code** (not the summary).
3. Present the raw code + the file's structural context (what it imports, what calls it) to the LLM.
4. Ask for a detailed, code-level opportunity analysis — citing specific functions, line ranges, and data flows.

**This is where the tool-based API comes in:**

```python
# Tools available to the deep-dive agent:
def get_module_tree(module_path: str) -> dict:
    """Returns the directory tree of a module with file summaries."""

def get_file_code(file_path: str) -> str:
    """Returns the full source code of a file."""

def get_function_code(file_path: str, function_name: str) -> str:
    """Returns the source code of a specific function."""

def get_callers(file_path: str, function_name: str) -> list:
    """Returns all functions that call this function (from the structural graph)."""

def get_callees(file_path: str, function_name: str) -> list:
    """Returns all functions called by this function."""

def search_symbols(query: str) -> list:
    """Fuzzy-search for symbols (classes, functions) by name across the repo."""
```

This is the PageIndex pattern: the agent navigates the pre-computed tree of summaries, then selectively fetches raw code only where it spots something interesting. The structural graph edges (`calls`, `imports`) let it trace data flows across files without reading everything.

---

## Part 4: Build New or Extend Existing?

**Build new.** Here's why:

1. **The existing scanner's architecture is fundamentally different.** It chunks files and calls the LLM per chunk with no persistent representation. Our design has three distinct layers (structural graph → semantic annotations → mining agent), each with different update frequencies and storage formats. Retrofitting this onto a chunk-and-call architecture would be harder than building fresh.

2. **The existing scanner re-reads everything every time.** The core value proposition of our design is incrementality — structural fingerprinting means most files are never re-analyzed. This requires a fundamentally different data flow.

3. **The existing scanner has no inter-file context.** It analyzes each chunk in isolation. Our design's key innovation is that the structural graph provides import/call context to the summarization prompt, and the mining agent can trace data flows across modules. This requires the structural graph to exist before summarization starts.

4. **The opportunity categories and report format can be reused.** The 11 categories (Personalization, Proactive Intelligence, NL Interfaces, etc.) are good and should be carried over. The Streamlit viewer concept should also be reused.

5. **The code is small enough that starting fresh is fast.** The core system is ~3 files: (a) a tree-sitter parser that builds the structural graph, (b) an LLM summarizer that annotates it, (c) a mining agent that queries it. This is a weekend project to get a v0, not a months-long rewrite.

**What to carry over from the existing scanner:**
- The 11 opportunity categories
- The per-module + master report structure
- The SQLite idea-store schema (for deduplicating opportunities across runs)
- The Streamlit viewer

---

## Part 5: Open Questions and Risks

### Things I'm confident about

- **Tree-sitter for structural parsing.** This is battle-tested in Aider (millions of users), RepoGraph, RANGER, and Understand-Anything. It's multi-language, fast, and accurate.
- **File-level LLM summaries.** Every system that produces useful codebase understanding (Understand-Anything, HCAG, PageIndex) does this. The cost is manageable with incremental updates.
- **JSON storage.** For repositories up to ~100k files, a JSON knowledge base loaded into memory is sufficient. We don't need a database.
- **The two-mode mining approach.** Broad scan using summaries + targeted deep-dive using raw code is the correct architecture.

### Things I'm less confident about — test on a real repo first

> [!WARNING]
> **1. Summary quality for opportunity mining.**
> The file summarization prompt asks the LLM to flag `ai_ml_signals` during summarization. I don't know if the LLM is good at spotting these patterns from a single file read. It might miss signals that only become apparent when you see the *combination* of multiple files (e.g., "this file reads from a rules table" + "this other file writes to that rules table" → "the rules table is a manual knowledge base that could be replaced by ML"). **Test:** Run the summarizer on 50 files from a known Java/Spring repo, manually evaluate the `ai_ml_signals` for recall.

> [!WARNING]
> **2. Directory summary quality at the top levels.**
> Recursive summarization (summarizing summaries) is known to lose detail. If the top-level project summary says "an e-commerce platform" but doesn't mention that there's a fraud detection module, the broad scan will miss it. **Test:** Generate directory summaries for a real 14-module repo, check if the top-level summary captures all major modules.

> [!WARNING]
> **3. Structural hash sensitivity.**
> Understand-Anything's fingerprinting strips comments and normalizes whitespace to compute a structural hash. But what counts as a "structural change" for opportunity mining? If someone changes a hardcoded threshold from `0.5` to `0.7`, the structural hash changes, but the opportunity is the same ("this threshold should be learned"). **Risk:** Unnecessary LLM re-summarization on trivial constant changes. **Mitigation:** Accept this as an acceptable cost — re-summarizing one file is ~$0.01.

> [!WARNING]
> **4. Tree-sitter query coverage across languages.**
> Aider ships tree-sitter query files (`.scm`) for ~30 languages. But these queries vary in quality — some languages have rich queries extracting classes, methods, imports; others only extract top-level definitions. **Test:** Run the parser on a real Java/Spring repo and verify it correctly extracts Spring annotations, dependency injection, REST endpoint mappings. If tree-sitter misses Spring-specific patterns, we may need custom queries.

> [!WARNING]
> **5. Scale of the broad scan prompt.**
> For a 14-module repo, the broad scan sends ~14 prompts to the LLM, each with ~2k tokens of summaries. For a 50-module monorepo, this becomes 50 prompts. **Test:** Run the broad scan on a large repo and verify the LLM doesn't lose context across modules (each module is scanned independently, but cross-module patterns need to be synthesized in a final aggregation step).

> [!CAUTION]
> **6. The "cold start" cost.**
> The first run on a 500-file repo requires 500 LLM calls for file summaries + ~50 directory summaries. At $0.01/file, this is $5-6. For a 5,000-file repo, it's $50-60. This is a one-time cost, but it's not trivial. **Mitigation:** (a) Use GPT-4o-mini or a similar cheap model for summarization. (b) Implement the HCAG compression-depth parameter — don't summarize test files, generated code, or vendored dependencies. (c) Allow the user to specify an ignore pattern (like `.gitignore` for the summarizer).

---

## Summary of What We Take From Each System

| System | What we take | What we skip |
|--------|-------------|-------------|
| **Aider/RepoMapper** | Tree-sitter parsing, PageRank importance scoring, token-budget binary search, diskcache pattern | The "active file personalization" (no interactive user) |
| **LocAgent** | 4-node/4-edge ontology (dir/file/class/func × contains/imports/calls/inherits) | Python-only AST parser, pickle serialization |
| **RepoGraph** | Stdlib/builtin filtering, tree-sitter query patterns | Flat node structure (no hierarchy) |
| **Understand-Anything** | Hybrid tree-sitter+LLM architecture, fingerprint-based incremental updates, JSON output schema, graph normalization | The dashboard/UI, tour generator, domain analyzer |
| **PageIndex** | Two-step retrieval (structure→content), vectorless reasoning-based navigation | PDF parsing, OCR pipeline |
| **HCAG** | Recursive directory summarization, compression-depth parameter | N/A (paper only) |
| **GraphRAG** | Community-level summarization concept | Leiden clustering, entity extraction, map-reduce query |
| **LightRAG** | Modular storage interfaces, incremental merge logic | Entity extraction prompts, vector search |
| **RAPTOR** | Recursive summarization concept | Embedding-based clustering |
| **CodexGraph/Code-Graph-RAG** | The idea of exposing the graph via queryable tools | Neo4j/Memgraph dependency, Cypher generation |
| **RANGER** | Two-path design (simple → direct, complex → exploration) | MCTS |
| **KGCompass** | Issue/PR to code entity linking (future: correlate Jira tickets with code) | Bug repair pipeline |
