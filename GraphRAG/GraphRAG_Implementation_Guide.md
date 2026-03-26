# GraphRAG — Lecture 3: Building GraphRAG in Practice

> **Series context:** This is Lecture 3 of a GraphRAG deep-dive.
> Lecture 1 covered RAG failure modes. Lecture 2 covered the GraphRAG architecture.
> This lecture is hands-on — setup, config, prompts, and common mistakes.

---

## The Landscape — Know Your Options

| Library | By | Best for |
|---|---|---|
| **microsoft/graphrag** | Microsoft Research | Production, full pipeline, most features |
| **LightRAG** | HKUDS | Simpler setup, faster indexing |
| **Neo4j + LlamaIndex** | Community | If you already use Neo4j |
| **nano-graphrag** | Community | Learning / understanding internals |

> This guide uses **Microsoft GraphRAG** — most battle-tested and maps 1:1 with the architecture in Lecture 2.

---

## Step 1 — Installation & Project Setup

```bash
# Create a clean virtual environment
python -m venv graphrag-env
source graphrag-env/bin/activate  # Windows: graphrag-env\Scripts\activate

pip install graphrag
```

```bash
# Create your working directory
mkdir my-graphrag && cd my-graphrag
mkdir -p ./input

# Drop your .txt documents into ./input/
# GraphRAG reads plain text — convert PDFs/docx beforehand

python -m graphrag init --root .
```

### What gets created

```
my-graphrag/
├── input/               # your documents go here
├── output/              # graph + summaries written here after indexing
├── prompts/             # ⭐ extraction prompts — tune these!
│   ├── entity_extraction.txt
│   ├── summarize_descriptions.txt
│   └── community_report.txt
└── settings.yaml        # main config file
```

> **Key insight:** The `prompts/` folder is where you spend 80% of your tuning time.

---

## Step 2 — The Settings File (`settings.yaml`)

```yaml
llm:
  api_key: ${GRAPHRAG_API_KEY}     # from env var
  type: openai_chat                # or azure_openai_chat
  model: gpt-4o                    # ⭐ use best model for extraction
  max_tokens: 4000

embeddings:
  llm:
    api_key: ${GRAPHRAG_API_KEY}
    type: openai_embedding
    model: text-embedding-3-small  # cheaper model is fine here

chunks:
  size: 1200                       # ⭐ tokens per chunk
  overlap: 100                     # overlap prevents edge splitting
  group_by_columns: [id]           # chunk per document

entity_extraction:
  prompt: prompts/entity_extraction.txt
  entity_types: [person, organization, drug, event]  # ⭐ domain-specific!
  max_gleanings: 1                 # re-prompt to catch missed entities

community_reports:
  prompt: prompts/community_report.txt
  max_length: 2000                 # tokens per community summary

cluster_graph:
  max_cluster_size: 10             # nodes per community (tune this)

local_search:
  text_unit_prop: 0.5              # ratio of context from chunks vs graph
  community_prop: 0.1
  top_k_mapped_entities: 10        # ⭐ how many entity hops to traverse

global_search:
  max_tokens: 12000
  data_max_tokens: 12000
  map_max_tokens: 1000             # tokens per community in map step
  reduce_max_tokens: 2000          # final synthesis tokens
```

### Two fields that matter most

**`entity_types`** — This is where most beginners leave performance on the table. The default is generic. Make it domain-specific:

| Domain | Example entity types |
|---|---|
| Biotech / pharma | `person, organization, drug, compound, trial, gene, outcome` |
| Legal | `person, organization, contract, clause, jurisdiction, case` |
| Finance | `person, organization, company, event, metric, regulation` |
| General | `person, organization, geo, event` ← default, often not enough |

**`max_gleanings`** — Re-prompts the LLM after initial extraction asking *"what did you miss?"* Set to `1` for most cases. Higher = better recall, more cost.

---

## Step 3 — The Extraction Prompt

This is the file that directly controls graph quality. Located at `prompts/entity_extraction.txt`.

### Anatomy of the prompt

**Part 1 — Goal (tune this for your domain)**
```
Given a text document that is potentially relevant to this activity and a
list of entity types, identify all entities of those types from the text
and all relationships among the identified entities.

→ Add your domain context here, e.g.:
"This is a biomedical research corpus. Focus on drugs, clinical outcomes,
researchers, and institutions."
```

**Part 2 — Entity format**
```
For each identified entity, extract the following information:
- entity_name: Name of the entity, capitalized
- entity_type: One of [{entity_types}]   ← your types injected here
- entity_description: Comprehensive description of the entity

Format each entity as:
("entity"{tuple_delimiter}"ENTITY_NAME"{tuple_delimiter}"TYPE"
{tuple_delimiter}"DESCRIPTION")
```

**Part 3 — Relationship format**
```
For each pair of related entities, extract:
- source_entity, target_entity
- relationship_description: explanation of why they are related
- relationship_strength: 1-10 score

("relationship"{tuple_delimiter}"SOURCE"{tuple_delimiter}"TARGET"
{tuple_delimiter}"DESCRIPTION"{tuple_delimiter}STRENGTH)
```

**Part 4 — Few-shot examples ⭐ (biggest quality lever)**

Add 2–3 examples from **your domain**. Generic examples = generic extraction.

```
Example input:
"Dr. Sarah Chen, board member of PharmaCo, oversaw the Phase 2 trial
of Compound X showing 34% efficacy improvement."

Example output:
("entity"..."DR. SARAH CHEN"..."PERSON"..."Researcher and board member of PharmaCo...")
("entity"..."PHARMACOCO"..."ORGANIZATION"..."Drug supplier and clinical trial sponsor...")
("entity"..."COMPOUND X"..."DRUG"..."Experimental compound in Phase 2 testing...")
("relationship"..."DR. SARAH CHEN"..."PHARMACOCO"..."board member of"...8)
("relationship"..."COMPOUND X"..."PHASE 2 TRIAL"..."tested in"...9)
```

---

## Step 4 — Running the Pipeline

### Index your corpus

```bash
python -m graphrag index --root .

# What happens under the hood:
# 1. Chunks all documents in ./input
# 2. Calls LLM extraction prompt on each chunk
# 3. Merges entities, deduplicates globally
# 4. Runs Leiden community detection
# 5. Calls LLM to write community summaries
# 6. Embeds everything, writes to ./output/

# Cost estimate: ~$5–20 per 1M tokens of input (GPT-4o)
# Time estimate: 10–30 min for a 100-doc corpus
```

### Run local search (fast — entity-anchored)

```bash
python -m graphrag query \
  --root . \
  --method local \
  --query "Who is connected to PharmaCo and what are their roles?"
```

Use local search for: specific entities, named people/orgs, multi-hop relationship questions.

### Run global search (slow — map-reduce over communities)

```bash
python -m graphrag query \
  --root . \
  --method global \
  --query "What are the main themes across all documents?"
```

Use global search for: thematic questions, corpus-wide summaries, "what are the main X" queries.

### Programmatic usage (for building a real app)

```python
from graphrag.query.api import local_search, global_search

result = await local_search(
    config_filepath="./settings.yaml",
    data_dir="./output",
    query="Who is connected to PharmaCo?",
    community_level=2   # 1=fine, 2=mid, 3=coarse communities
)

print(result.response)      # the answer
print(result.context_data)  # what was retrieved — inspect this!
```

### Community level guide

| Level | Granularity | Use when |
|---|---|---|
| `1` | Fine-grained, specific clusters | Precise entity questions |
| `2` | Mid-level (default) | Most queries |
| `3` | Broad, thematic clusters | High-level summary questions |

Think of it like zoom levels on a map — don't always zoom to street level.

---

## Step 5 — Inspecting the Output

**Before querying, always inspect the graph.** The output parquet files are your ground truth.

```python
import pandas as pd

# Check entity quality
entities = pd.read_parquet("./output/artifacts/create_final_entities.parquet")
print(entities[["title", "type", "description"]].head(20))

# Check relationships
relations = pd.read_parquet("./output/artifacts/create_final_relationships.parquet")
print(relations[["source", "target", "description", "weight"]].head(20))

# Check community summaries
communities = pd.read_parquet("./output/artifacts/create_final_community_reports.parquet")
print(communities[["title", "summary"]].head(10))
```

**Signs of bad extraction to look for:**
- Same entity under multiple names: `"Dr. Chen"`, `"Sarah Chen"`, `"S. Chen"` → should be one node
- Vague relationship descriptions: `"is related to"` → should be specific
- Missing entities you expected → extraction prompt needs domain tuning
- Hallucinated relationships → add negative examples to your prompt

---

## The 5 Mistakes Everyone Makes

### Mistake 1 — Generic entity types for a specialized domain
```yaml
# ✗ Default — too generic
entity_types: [person, organization, geo, event]

# ✓ Domain-specific — what you actually need
entity_types: [researcher, drug, trial, institution, compound, outcome]
```
The LLM only extracts what you tell it to look for.

### Mistake 2 — Not inspecting the graph before querying
```
✗  Index → query → confused by bad answers → don't know why

✓  Index → open output parquet files → verify entity quality → then query
```
Bad answers almost always trace back to bad extraction. Check the graph first.

### Mistake 3 — Using global search for specific questions
```
✗  global search: "What did the CEO say about Q3?" → slow, vague
✓  local search:  "What did the CEO say about Q3?" → fast, precise

✗  local search:  "What are the main themes?" → misses big picture
✓  global search: "What are the main themes?" → correct
```
In production, use an LLM classifier to route queries automatically.

### Mistake 4 — Chunks too small
```yaml
# ✗ Too small — entity and its context end up in different chunks
size: 300

# ✓ Large enough to keep entity + relationship context together
size: 1000  # to 1500
overlap: 100  # to 200
```
The extraction LLM needs to see the entity AND its context in the same chunk.

### Mistake 5 — Ignoring community_level at query time
```python
# ✗ Always fine-grained — misses broader themes
community_level=1

# ✓ Match level to query type
community_level=1   # "Tell me about PharmaCo's board"
community_level=2   # most queries (default)
community_level=3   # "What are the major risk themes across all documents?"
```

---

## Recommended Order of Operations When Starting

1. **Start small** — 10–20 documents, not your full corpus. Validate quality cheaply first.
2. **Tune `entity_types`** for your domain before running anything.
3. **Add few-shot examples** to the extraction prompt using your actual data.
4. **Run indexing** and inspect the parquet output before any queries.
5. **Test local and global search** with representative queries from each category.
6. **Scale up** only after the small-scale graph looks clean.

---

## Quick Reference Cheat Sheet

| Question type | Strategy | Why |
|---|---|---|
| "Who is X?" | Local | Entity lookup |
| "How does X relate to Y?" | Local | Edge traversal |
| "What did X say about Y?" | Local | Entity + claim retrieval |
| "What are the main themes?" | Global | Community summary sweep |
| "Summarize the corpus" | Global | Map-reduce across communities |
| "What risks exist across all docs?" | Global | Thematic, corpus-wide |
| "Find conflicts of interest" | Local | Multi-hop graph traversal |

---

## Resources

- [Microsoft GraphRAG GitHub](https://github.com/microsoft/graphrag)
- [Official docs](https://microsoft.github.io/graphrag/)
- [Original paper: "From Local to Global"](https://arxiv.org/abs/2404.16130) — Edge et al., 2024
- [LightRAG](https://github.com/HKUDS/LightRAG) — simpler alternative
- [nano-graphrag](https://github.com/gusye1234/nano-graphrag) — for understanding internals

---

*Notes from a GraphRAG deep-dive lecture series. Lecture 1: RAG failure modes. Lecture 2: GraphRAG architecture. Lecture 3: Building in practice (this doc).*
