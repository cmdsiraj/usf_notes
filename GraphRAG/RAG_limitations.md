# Why RAG Breaks — The Real Problems
### The Mental Model of Vanilla RAG
> **Vanilla RAG** = Split documents into chunks → Embed chunks → Store in vector DB → At query time, embed the question → Retrieve top-K similar chunks → Stuff into LLM context → Generate answer.
Simple, powerful, and... surprisingly fragile. Here's where it breaks:

## Failure Mode 1: The "Chunk Isolation" Problem
When you chunk a document, each chunk is a semantically sealed unit. The vector embedding captures the local meaning of that chunk — but has **zero awareness of what's in other chunks.** At retrieval time, you're betting that your top-K chunks happen to contain everything needed. That bet fails constantly.
Let me show you a concrete scenario:
**Scenario:** You have a 200-page financial report. A user asks: "What was the strategic reason the CEO gave for the Q3 revenue drop, and how does it connect to the supply chain issues mentioned in the risk section?"
This question requires synthesizing information from at least 3 different sections — the CEO letter (page 2), the revenue breakdown (page 47), and the risk factors (page 130). Your vector search will likely retrieve chunks that are semantically similar to the query... but they may not be the right 3 chunks, and even if they are, **they arrive with no structural relationship between them.**
<img width="857" height="520" alt="image" src="https://github.com/user-attachments/assets/dcf4c80f-32bf-497d-9736-0fe167a53d55" />
The key insight: **chunks are retrieved by semantic similarity to the query, not by their logical relationship to each other.** The CEO letter and the supply chain risk section might not be semantically close to each other or to the query — so they both miss the top-K cut.

## Failure Mode 2: Keyword/Semantic Mismatch
The classic example: the document says "the medication caused adverse hepatic outcomes" but the user asks "did the drug damage the liver?" These mean the same thing, but in embedding space they may not be close enough. This is partly why dense retrieval was a big deal — but even modern embeddings struggle with domain jargon inversions.

## Failure Mode 3: Multi-Hop Reasoning (The one you didn't flag — and the most important one)
This is the hardest and most fundamental RAG limitation.
**Scenario:** You're building a knowledge base about a biotech company. A user asks: "Is there any conflict of interest between our lead researcher and any of our drug suppliers?"
To answer this, the system needs to:
1. Find who the lead researcher is → Dr. Sarah Chen
2. Find her disclosed affiliations → board member of PharmaCo Ltd.
3. Find our drug suppliers → includes PharmaCo Ltd.
4. Connect step 2 and step 3 → **conflict of interest exists**
No single chunk contains this answer. The answer emerges from traversing a chain of relationships. Vanilla RAG **cannot do this** — it has no mechanism to follow relationship chains. It can only retrieve chunks that are directly similar to the query text.

## Failure Mode 4: The "What's the Big Picture?" Problem (The other one you didn't flag)
Ask vanilla RAG: "What are the main themes across all 500 customer support tickets this month?"
It will retrieve the top-K tickets most similar to the word "themes" — which is meaningless. There is no chunk anywhere in your index that contains a summary of all 500 tickets, because **you never wrote one.** Vanilla RAG can only retrieve what exists. It cannot synthesize global summaries from a corpus. This is the **global query problem.**

___
Now let me show you the full RAG failure landscape together:
<img width="892" height="440" alt="image" src="https://github.com/user-attachments/assets/09d4deac-1bec-49e1-97bc-580e900b2244" />

