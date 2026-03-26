# GraphRAG Architecture — From the Ground Up
Here's the core idea in one sentence:
> GraphRAG = Extract entities & relationships from your corpus → Build a knowledge graph → At query time, use the graph structure to retrieve richer, more connected context than chunks alone could ever give you.
___
> Chunks → LLM extraction → Graph nodes & edges → Community detection → Community summaries
___

There are two distinct phases. Let's walk through them.
## Phase 1: The Indexing Pipeline
This is where all the heavy lifting happens — offline, before any user asks a single question. This is also what makes GraphRAG expensive to set up but incredibly powerful at query time.

### Step 1 — Text chunking (same as vanilla RAG)
Nothing new here. You split your documents into chunks. GraphRAG still does this — chunks are the source material, they just stop being the unit of retrieval.

### Step 2 — Entity & Relationship Extraction (the magic step)
This is where GraphRAG diverges radically from vanilla RAG. Instead of just embedding your chunks, you run an LLM over each chunk to extract:

- **Entities —** named things: people, organizations, concepts, locations, events
- **Relationships —** how those entities connect: "Dr. Chen is a board member of PharmaCo", "PharmaCo supplies compound X"
- **Claims —** factual assertions attached to entities: "Q3 revenue dropped 18%"

Think of this as turning prose into structured knowledge.

<img width="1440" height="762" alt="image" src="https://github.com/user-attachments/assets/13d96f48-de41-4433-ae28-3ccdbf2255ac" />

### Step 3 — Graph Construction
All those extracted entities and relationships get merged into a single knowledge graph. Same entity mentioned in 100 different chunks? It's one node in the graph, with 100 edges pointing back to the source chunks. This deduplication and merging is where the real power comes from.

### Step 4 — Community Detection (GraphRAG's secret weapon for global queries)
This is the step that vanilla RAG has no equivalent of. After the graph is built, a community detection algorithm (typically **Leiden algorithm**) partitions the graph into clusters of highly-connected nodes — called **communities.**

Then, an LLM generates a **summary for each community**. These summaries are stored and indexed separately.

This is how GraphRAG answers global queries like "what are the main themes?" — it queries the community summaries, not the raw chunks.

Now let me show you the full indexing pipeline end-to-end:

<img width="1440" height="1016" alt="image" src="https://github.com/user-attachments/assets/22e4ac44-90c3-4bd7-aace-a1a102f97afb" />

## Phase 2: Query Time — Local vs. Global Search
This is where GraphRAG gets elegant. It has **two fundamentally different retrieval strategies**, and choosing between them is a design decision you'll make when building.

**Local search —** for specific, entity-focused questions. "What did the CEO say about supply chains?" → find the CEO entity → traverse its edges → pull connected chunks → answer.

**Global search —** for broad, thematic questions. "What are the main risks across all our documents?" → query community summaries → synthesize across summaries → answer.

<img width="1440" height="910" alt="image" src="https://github.com/user-attachments/assets/e6c77423-d24a-4cfb-9f4f-f9c3464fb45a" />

<img width="889" height="451" alt="image" src="https://github.com/user-attachments/assets/3bb4c6a8-19fd-4044-adaa-42ceeba2ab7e" />

___
> **GraphRAG is not always the right tool.** If your queries are mostly specific factual lookups and your corpus is small, vanilla RAG is fine. GraphRAG earns its keep when your corpus is large, relationship-dense, and users ask complex cross-document questions.

