# Query 6

## Result

**Original Query:** 我是一个很精通RAG技术的工程师，现在老板让我做一个关于代码转换的工作，类似于自然语言转化为SQL查询语言，希望借助历史转化数据，自动实现新的代码转换。具体而言，这个代码转化工作，你可以理解为一个专业厂商定制的OPC语言（类似于芯片DRC规则语言），一般来说会有一个表格字典，描述大部分规则，他的键值有id，command，rank，describe等。然后需要根据这个规则书写我的OPC代码/DRC代码，也就是 `X command rank target` 这种，去描述我要量测或者计算某块芯片的面积等等功能。我现在希望你给我一个详细的技术方案，如何去做规则解析，如何利用RAG技术辅助大模型做这个OPC代码的转换，以及其中的原理或者需要我去深度研究的地方。

**LLM answer:**

This is a well-scoped problem that maps cleanly onto the RAG-for-code-generation paradigm (most famously demonstrated by Text-to-SQL research). Below is a detailed technical plan covering rule parsing, knowledge-base design, the RAG pipeline, prompt engineering, and the areas you will need to study most deeply.

---

### 1. Understand and Parse the Rule Dictionary

The rule dictionary (table with columns like `id`, `command`, `rank`, `describe`, and possibly `syntax`, `operand_types`, `examples`) is your primary structured knowledge source.

**Step 1 — Extract the schema precisely**

Load the table (CSV / Excel / proprietary format) and normalise every row into a canonical JSON record:

```json
{
  "id": "M001",
  "command": "AREA",
  "rank": 2,
  "describe": "Computes the geometric area of the target layer polygon.",
  "syntax": "AREA <layer> [OPTIONS]",
  "operand_types": ["layer"],
  "examples": ["AREA metal1", "AREA poly INSIDE active"]
}
```

**Step 2 — Enrich each rule with a natural-language description**

Use an LLM (GPT-4o / Claude 3.5) to expand terse `describe` fields into a full paragraph that explains: what the command does, what inputs it accepts, common use cases, and common errors. Store this enriched text alongside the original record. This dramatically improves embedding quality because short, jargon-heavy descriptions embed poorly.

**Step 3 — Build a semantic rule index**

Embed each enriched record (command name + description + examples joined as a single string) with a code-aware embedding model (e.g. `text-embedding-3-large`, `voyage-code-2`, or a locally fine-tuned model). Store vectors in Qdrant, Weaviate, or pgvector.

---

### 2. Build the Historical Conversion Corpus

Your historical examples (natural-language intent → OPC code pairs) are your second, and arguably most important, knowledge source. This mirrors few-shot example retrieval in Text-to-SQL systems like DIN-SQL and DAIL-SQL.

**Data format for each example:**

```json
{
  "id": "EX-042",
  "intent": "计算 metal1 层在 active 区域内部的总面积",
  "opc_code": "AREA metal1 INSIDE active",
  "rules_used": ["M001"],
  "tags": ["area", "metal1", "inside", "layer-operation"]
}
```

**Embedding strategy:** embed both the `intent` field and the `opc_code` field. At query time, you match the user's natural-language description against `intent` embeddings. This is essential because OPC code syntax is semantically distant from natural language; matching on intent avoids the vocabulary gap.

**Minimum corpus size:** plan for at least 200–500 well-labelled examples before the system becomes reliable. The Text-to-SQL literature shows diminishing returns per example beyond ~1,000 for a fixed schema.

---

### 3. RAG Pipeline Architecture

```
User NL intent
     │
     ▼
┌─────────────────────────────┐
│  Query Optimizer (LLM)      │  ← Rewrite / decompose the intent into
│  - extract target layers     │    structured sub-queries
│  - identify operations       │
└─────────────┬───────────────┘
              │  structured sub-queries
     ┌────────┴────────┐
     ▼                 ▼
┌──────────────┐  ┌──────────────┐
│ Rule Index   │  │ Example Index│
│ (Qdrant)     │  │ (Qdrant)     │
└──────┬───────┘  └──────┬───────┘
       └────────┬─────────┘
                ▼
        ┌──────────────┐
        │  Reranker    │  ← Cohere rerank-v3 / bge-reranker
        └──────┬───────┘
               ▼
      ┌─────────────────┐
      │  Context Builder│  ← Assemble: rules + examples + schema
      └──────┬──────────┘
             ▼
      ┌─────────────────┐
      │  LLM Generator  │  ← GPT-4o / Qwen-Coder / DeepSeek-Coder
      └──────┬──────────┘
             ▼
      ┌─────────────────┐
      │  Syntax Checker │  ← Parse generated code against grammar
      └──────┬──────────┘
             ▼
        Final OPC Code
```

**Key design choices:**

- **Hybrid retrieval:** Use both sparse (BM25) and dense retrieval on the example index. Command names like `AREA`, `EDGE`, `INSIDE` are exact-match tokens that BM25 handles well; semantic intent is handled by dense vectors.
- **Dual-index strategy:** retrieve from the rule index and the example index in parallel, then merge and rerank. This ensures the model always has both the formal rule specification and concrete usage examples.
- **Top-k tuning:** start with top-5 rules + top-3 examples per query. Increase if generated code frequently uses the wrong command variant.

---

### 4. Prompt Engineering for OPC Code Generation

The final prompt sent to the LLM should contain four sections:

```
## Task
You are an OPC/DRC code generation assistant. Given a natural-language description 
of a chip measurement or geometric operation, generate the correct OPC code using 
ONLY the commands defined in the rule dictionary below.

## Rule Dictionary (retrieved)
[RULE M001] AREA — Computes the geometric area of the target layer polygon.
  Syntax: AREA <layer> [INSIDE <layer>] [OPTIONS]
  ...

## Similar Examples (retrieved)
[EXAMPLE 1]
  Intent: 计算 metal1 层在 active 区域内的总面积
  OPC Code: AREA metal1 INSIDE active

[EXAMPLE 2]
  Intent: 量测 poly 层边缘到 nwell 的最小间距
  OPC Code: EXTCHK poly nwell MIN 0.18

## User Request
{user_intent}

## Output Format
Return ONLY valid OPC code, one statement per line. Do not explain.
```

**Critical constraints to enforce:**
- Instruct the LLM to use only commands present in the retrieved rule dictionary (prevents hallucinated commands).
- Ask for chain-of-thought reasoning in a scratchpad, then emit only the final code (use `<think>` / `<answer>` tags if the model supports them, e.g., DeepSeek-R1).
- Include the OPC grammar's formal EBNF if it is small enough to fit in the context window.

---

### 5. Syntax Validation Layer

After LLM generation, pass the output through a lightweight syntax checker before returning it to the user. This can be:

1. **Grammar-based parser** (ANTLR or lark-parser): write a formal grammar for the OPC dialect if you have access to the specification. The parser will catch structurally invalid statements immediately.
2. **Rule-table validator**: check that every command token in the generated code exists as a `command` value in your rule dictionary.
3. **LLM self-critique loop**: if syntax validation fails, feed the error back to the LLM with the message "The following statement is invalid: `<stmt>`. The error is: `<error>`. Please correct it." Limit to 2–3 retry loops.

---

### 6. Fine-Tuning Path (for later iterations)

Once you have 1,000+ validated (intent, OPC code) pairs, consider supervised fine-tuning a smaller code-specialized model (Qwen2.5-Coder-7B, StarCoder2-7B) on your corpus. Fine-tuning allows the model to internalize OPC syntax, reducing reliance on in-context examples and improving latency. RAG can remain as a complement, providing runtime access to the latest rule updates without requiring retraining.

---

### 7. Evaluation Framework

Mirror the evaluation methodology from Text-to-SQL benchmarks:

| Metric | Description |
|--------|-------------|
| **Exact Match (EM)** | Generated code is character-identical to the gold reference |
| **Syntax Validity** | Generated code parses successfully against the OPC grammar and rule table |
| **Semantic Correctness** | Syntactically valid code also produces the correct output on a test chip layout (distinct from syntax — a statement can parse but compute the wrong result) |
| **Component Match** | Command, operands, and options individually correct (partial credit) |
| **Human Review Rate** | Percentage of outputs requiring human correction in production |

Build a regression test suite from your historical examples and run it on every model/prompt update.

---

### 8. Key Areas to Study Deeply

| Topic | Why it matters for this task |
|-------|------------------------------|
| **Text-to-SQL (DAIL-SQL, DIN-SQL, CHESS)** | Directly analogous problem; most retrieval and prompt engineering insights transfer |
| **Code RAG & code embeddings** (voyage-code-2, CodeBERT) | OPC code has specific token distributions; general text embeddings underperform |
| **Few-shot example selection** (DAIL-SQL masked question similarity) | Which examples to retrieve has the biggest single impact on accuracy |
| **Structured data ingestion into RAG** | Your rule table is structured; research on table-to-text and schema-linking is relevant |
| **Grammar-constrained decoding** (Guidance, LMQL, Outlines) | Constrain LLM output to only syntactically valid OPC tokens — eliminates whole classes of errors |
| **RAFT (Retrieval-Augmented Fine-Tuning)** | Fine-tune with distractor rules mixed in to make the model robust to imperfect retrieval |
| **Agentic/iterative retrieval** (Self-RAG, ReAct) | For complex multi-statement OPC programs, multi-step retrieval with self-correction outperforms single-pass RAG |

---

**Summary of recommended stack:**

| Component | Recommended tool |
|-----------|-----------------|
| Vector store | Qdrant (hybrid BM25 + dense) |
| Embedding model | `voyage-code-2` or `text-embedding-3-large` |
| Reranker | Cohere `rerank-v3` or `bge-reranker-v2-m3` |
| LLM generator | GPT-4o or Qwen2.5-Coder-32B |
| Syntax checker | lark-parser + rule-table validator |
| Orchestration | LangChain / LlamaIndex |
| Evaluation | Custom harness inspired by Spider/BIRD benchmark tooling |

---

### RAG Pipeline

**Latency Breakdown (estimated):**

- Query optimizer: ~1.5s
- Retrieve (dual-index, **runs in parallel**): ~0.8s
- Rerank: ~0.6s
- Context assembly: ~0.2s
- LLM generation: ~5–15s (depends on model and code length)
- Syntax check + retry (if needed): ~0–3s

**Total end-to-end:** ~8–21s. The dual-index retrieval (rule index + example index) runs in parallel; query optimisation and Qdrant client warm-up also overlap. The dominant cost is LLM generation.

**Query Optimizer**

**Tech:** GPT-4o-mini, structured responses

For a user input like *"计算 metal1 层在 active 区域内部的总面积"*, the optimizer produces:
1. **Primary operation:** area calculation
2. **Layers involved:** metal1, active
3. **Spatial relationship:** inside
4. **Search queries:** `AREA command layer operation`, `INSIDE spatial filter OPC`
5. **Search method:** hybrid

**Retrieve**

**Tech:** Qdrant, hybrid (sparse BM25 + dense)

Dual-index retrieval:
- **Rule index** top-5 for query: *area calculation layer chip OPC command*
- **Example index** top-5 for query: *计算 metal1 层在 active 区域内部的总面积*

**Rerank**

**Tech:** Cohere API `rerank-v3` (or `bge-reranker-v2-m3` for on-premise)

Merge and rerank all candidates, keep top-3 rules + top-3 examples with highest cross-encoder scores.

**Context Assembly**

Concatenate: retrieved rules + retrieved examples + user intent into the structured prompt template from Section 4.

**LLM Generation + Syntax Check**

Generate OPC code, validate against grammar and rule table, retry up to 3× on failure.
