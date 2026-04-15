# Graph-Rag-for-wellness-and-mindfulness
A GraphRAG system built on wellness and hypnotherapy documents that extracts entities, relationships, and attributes into a knowledge graph — then compares graph-based retrieval against standard RAG on identical queries.

## What This Project Builds

A complete GraphRAG pipeline that:
- Extracts entities, relationships, and structured attributes from 4 PDFs using GPT-4o-mini
- Stores them in a NetworkX knowledge graph with 709 nodes and 542 edges
- Queries the graph using hybrid keyword matching and graph traversal
- Generates grounded LLM answers from graph context
- Falls back to hybrid RAG (OpenAI embeddings + BM25) when graph context is insufficient
- Compares GraphRAG answers against a production-grade standard RAG baseline on 10 identical questions

- ## The Core Finding

**GraphRAG quality is 80% determined by how well the knowledge graph was built. Everything else — querying, retrieval, LLM prompting — is just plumbing.**

In our evaluation across 10 questions:
- GraphRAG won on 2 questions — breathing techniques and children's roles in dysfunctional families
- Standard RAG won on 6 questions — factual, self-contained questions where chunk retrieval finds complete passages
- 2 questions were ties

The pattern was clear: GraphRAG won when the graph had comprehensive, well-structured attribute data for the queried entity. It lost when extraction missed content that existed in the raw PDFs. This is the finding that most GraphRAG tutorials skip — because they use controlled datasets.

---

## Architecture
PDF Documents (4 files, ~465K characters)
↓
Chunking (583 chunks, 1000 chars, 200 overlap)
↓
Entity + Relationship + Attribute Extraction (GPT-4o-mini, 583 API calls)
↓
Cleaning (remove noise, merge attributes across chunks)
↓
NetworkX Knowledge Graph (709 nodes, 542 edges)
↓
Query Pipeline:
├── Query Classifier (clear/vague, simple/complex)
├── Query Rewriter (uses last 4 conversation turns if vague)
├── Keyword Extractor (compound phrase aware)
├── Hybrid Node Matcher (3 passes: compound, type-aware, semantic)
├── Context Retriever (per-node budget, priority nodes first)
├── Context Quality Check (pre-generation gate)
├── LLM Answer Generator (temperature=0)
├── LLM Self-Assessment (post-generation gate)
└── RAG Fallback (hybrid search: OpenAI embeddings + BM25)

## Knowledge Graph Schema

### Entity Types
| Type | Examples |
|---|---|
| CONCEPT | Trance, Consciousness, Suggestibility |
| TECHNIQUE | Pranayama, Progressive Relaxation, Eye Fixation |
| STATE | Hypnotic State, Meditative State, Deep Sleep |
| CHAKRA | Root Chakra, Throat Chakra, Ajna |
| PRACTICE | Yoga, Hypnotherapy, Meditation |
| CONDITION | Anxiety, Phobia, Trauma, Depression |
| EMOTION | Fear, Anger, Embarrassment, Grief |
| BODY_PART | Nervous System, Diaphragm, Breath |

### Relationship Types
| Relationship | Example |
|---|---|
| INDUCES | Pranayama → INDUCES → Meditative State |
| TREATS | Hypnotherapy → TREATS → Anxiety |
| PART_OF | Pranayama → PART_OF → Yoga |
| ACTIVATES | Breathwork → ACTIVATES → Root Chakra |
| REQUIRES | Ericksonian Hypnosis → REQUIRES → Rapport |
| LEADS_TO | Deep Relaxation → LEADS_TO → Trance |
| SIMILAR_TO | Hypnotic State → SIMILAR_TO → Meditative State |
| DEVELOPED_BY | Ericksonian Hypnosis → DEVELOPED_BY → Milton Erickson |
| CONTRAINDICATED_FOR | Deep Hypnosis → CONTRAINDICATED_FOR → Psychosis |

### Node Attributes (new vs standard GraphRAG)
Each entity type carries structured attributes extracted during ingestion:
- CHAKRA: affirmations, element, color, body_location, associated_qualities
- TECHNIQUE: steps, purpose, duration, benefits
- PRACTICE: benefits, contraindications, origin
- CONDITION: symptoms, causes, related_emotions
- STATE: characteristics, how_to_achieve, duration
- CONCEPT: definition, related_principles
- EMOTION: physical_sensations, triggers, related_conditions
- BODY_PART: functions, related_systems, associated_practices

## Key Engineering Decisions

### 1. Schema design is the hardest part
Entity types, relationship types, and node attributes must be defined by a human with domain understanding before extraction begins. The LLM finds instances within those categories but cannot define the categories itself. This is the core knowledge engineering bottleneck of GraphRAG — and the reason it is expensive to build and maintain in production.

### 2. Node attributes are not optional
List-style content — affirmations, steps, symptoms, benefits — does not extract into entity-relationship pairs. Without explicit attribute schema design and extraction, a GraphRAG system cannot answer questions like "what are the affirmations of X?" or "what are the steps of Y?" We discovered this gap mid-build when the Throat Chakra query returned no affirmations despite having an affirmation node.

### 3. Attribute merging across chunks is essential
The same entity appears in multiple chunks. Each chunk contributes partial attributes. The graph must merge attributes across all occurrences — not overwrite. Without merging, Root Chakra would have only one affirmation instead of four.

### 4. Three-pass hybrid keyword matching
- Pass 0: Compound phrase matching against full query text
- Pass 0b: Type-aware category search (triggered by "types of", "different types", etc.)
- Pass 1: Per-keyword string matching with exact, partial, word-level scoring (word length > 4 to avoid filler words)
- Pass 2: Semantic embedding fallback via sentence-transformers

### 5. Per-node traversal budget prevents context domination
With a shared traversal pool, highly connected nodes consume all available slots and crowd out other matched nodes. Each matched node gets an equal share of traversal slots and traverses independently. For a breathing query with 17 matched nodes and max_nodes=40: budget = 1 slot each. Progressive Relaxation cannot dominate.

### 6. Two-layer fallback protection
- Pre-generation gate: checks node count, edge count, context length before making LLM call
- Post-generation gate: LLM self-assessment after answer generation
- Fallback uses original query (not rewritten query) for chunk retrieval — more precise
- Fallback uses RAG hybrid search (OpenAI embeddings + BM25) — same model as baseline

### 7. Temperature = 0 for all LLM calls
Creativity is the enemy of faithfulness in retrieval-grounded systems. Temperature 0.3 allowed the LLM to generate affirmations not present in source documents. Temperature 0 ensures answers are grounded strictly in retrieved context and reproducible across runs.

### 8. Query rewriting is conditional
The system classifies clarity and complexity before rewriting. A query is vague only if it contains pronouns or cannot be understood standalone. Running rewriting on every query distorts already precise queries and produces over-engineered rewritten versions that match wrong graph neighborhoods.

---

## When GraphRAG Makes Sense

Use GraphRAG when:
- Data is highly interconnected with meaningful typed relationships
- Queries require multi-hop reasoning across concepts
- Knowledge base is relatively stable — does not change daily
- Domain is narrow and well-defined enough for schema design upfront
- A team exists to maintain and iterate on the schema over time

Use Standard RAG when:
- High velocity data that changes constantly
- Broad consumer products with unpredictable user queries
- Queries are mostly "find me similar content"
- No knowledge engineering team available
- Quick deployment is needed

### The B2C Reality
For a B2C product with daily user interactions and constantly changing data, a full knowledge graph as the primary data store is impractical. GraphRAG is better suited to the stable knowledge layer — domain concepts, clinical guidelines, product relationships — not the dynamic user interaction layer.

### Organizations are not replacing databases with graphs
Companies add a graph layer on top of existing infrastructure. Tables handle transactional data. Graphs handle relationship reasoning tables cannot express. LLMs are now making this cheaper — historically graphs required expensive manual curation, now LLMs extract entities and relationships automatically.

## Standard RAG Baseline Performance (RAGAS Evaluation)

| Metric | Score |
|---|---|
| Context Precision | 0.94 |
| Faithfulness | 0.88 |
| Context Recall | 0.88 |
| Answer Relevancy | 0.81 |

The baseline uses: LangChain + OpenAI text-embedding-3-small + FAISS + BM25 hybrid search + GPT-4o-mini + query rewriting + conversation memory.

## GraphRAG vs Standard RAG — Side by Side Results

| Question | Type | Winner | Reason |
|---|---|---|---|
| Corrective therapy | Factual | RAG (slight) | RAG retrieved complete passage including reframing/future pacing |
| Two forms of anxiety | Factual | RAG | RAG returned exact terminology; GraphRAG used informal phrasing |
| Dysfunctional family traits | Factual | RAG | RAG retrieved 13 traits; GraphRAG returned 9 (graph gap) |
| When dysfunctional families created | Factual | Tie | Identical answers |
| Healthy family traits | Factual | Tie | GraphRAG richer descriptions; RAG cleaner list |
| Children's roles | Factual/List | GraphRAG | GraphRAG returned all 4 roles; RAG only returned 2 |
| Breathing types | Category | GraphRAG | GraphRAG returned 6 techniques with steps and benefits; RAG returned 3 without detail |
| Throat chakra | Relational | RAG | RAG retrieved richer passage; graph had incomplete attribute data |
| Throat chakra affirmations | Follow-up | RAG | RAG retrieved 4 affirmations; graph only had 1 stored |
| Law of repetition | Factual | RAG | RAG answer directly grounded; GraphRAG added generic framing |

**GraphRAG won: 2/10. Standard RAG won: 6/10. Ties: 2/10.**

--

## Objective Evaluation Results (LLM-as-Judge)

Evaluated both systems on the same 10 questions using GPT-4o-mini as judge.
Each answer scored on Faithfulness, Completeness, and Relevance (1-5 scale).

| Metric | GraphRAG | Standard RAG |
|---|---|---|
| Faithfulness | 4.10 | 4.10 |
| Completeness | 4.00 | 3.70 |
| Relevance | 4.80 | 4.90 |
| Overall | 4.30 | 4.23 |

Per-question winners:
- GraphRAG won: Q4 (dysfunctional family creation), Q6 (children roles), 
  Q7 (breathing techniques), Q10 (law of repetition)
- RAG won: Q1 (corrective therapy), Q2 (anxiety forms), Q3 (dysfunctional family traits)
- Tied: Q5 (healthy family), Q8 (throat chakra), Q9 (affirmations)

Key finding: GraphRAG's advantage is in Completeness — it retrieves more  comprehensive answers on category and list queries. Faithfulness is identical  across both systems. RAG has a slight edge on Relevance — answers are more concise.

---

## Key Learnings

1. **The graph is everything.** GraphRAG quality is almost entirely determined by extraction quality. A well-built graph with comprehensive attributes and accurate relationships will outperform standard RAG on structured queries. A graph with gaps will lose to RAG every time.

2. **Schema design requires domain expertise.** You cannot delegate this to the LLM. A human must decide what kinds of entities matter, what relationships are meaningful, and what attributes each entity type should carry. This is the most intellectually demanding part of the project.

3. **Attributes are not a nice-to-have.** A graph without node attributes can only answer relational questions. A graph with attributes can answer both relational and descriptive questions. Most GraphRAG tutorials skip attributes — this is a significant omission.

4. **Chunk retrieval still wins on factual questions.** For self-contained factual questions where the answer is a complete passage in the document, similarity search finds it directly. Graph traversal adds overhead without adding value. The optimal production system uses both.

5. **Extraction is a 15-minute, $1 investment you make once.** Once the graph is built, query-time cost is comparable to standard RAG. The upfront cost is the real difference.

6. **GraphRAG maintenance is ongoing.** Schema changes, new document ingestion, attribute updates, bad edge removal — all require human judgment. This is why GraphRAG is expensive to maintain and unsuitable for domains where data changes rapidly.

7. **Temperature=0 in retrieval systems.** Any temperature above 0 allows the LLM to generate plausible-sounding content not grounded in retrieved context. In a retrieval system, faithfulness to context is always more important than creative fluency.

---

## Built With

- Python + Google Colab
- OpenAI GPT-4o-mini (extraction + generation)
- NetworkX (graph storage and traversal)
- PyVis (interactive graph visualization)
- LangChain + FAISS + BM25 (RAG baseline)
- Sentence Transformers (semantic node matching)
