# 04 · RAG

Retrieval-Augmented Generation is the topic where entry-level candidates get caught out hardest — not because the mechanics are obscure, but because most self-taught RAG knowledge comes from building a single toy pipeline over a PDF, not from operating one in production against messy, multi-tenant, constantly-updating data. This file leans deliberately toward the production-scale failure modes: diagnosing whether a bad answer is a retrieval problem or a generation problem, handling documents that change, and access control across tenants — because that's where the real interview signal is, and where most candidates at this level have the least lived experience.

---

## Multiple choice

### Q4.1 · RAG vs fine-tuning · [Recall]

A team wants their support chatbot to answer questions using their internal knowledge base, which changes daily as new articles are published. What's the strongest argument for RAG over fine-tuning here?

- **A.** RAG produces more fluent, natural-sounding responses than a fine-tuned model
- **B.** Fine-tuning can't be done on domain-specific text, only RAG can incorporate it
- **C.** RAG lets you update the knowledge source without retraining, so daily-changing content stays current
- **D.** RAG is always cheaper to run at inference time than a fine-tuned model

<details>
<summary>Answer</summary>

**C**

**Why C:** The core tradeoff is *how knowledge gets updated*. Fine-tuning bakes knowledge into weights — updating it means retraining or re-tuning, which is slow and expensive to do daily. RAG keeps knowledge in an external, swappable index — you re-embed and re-index the new article, and the next query picks it up immediately. For daily-changing content, that update latency is the whole ballgame.

**Why not A:** Fluency is a property of the base model's generation quality, not of whether you used RAG or fine-tuning. Neither technique inherently makes prose more natural.

**Why not B:** Fine-tuning absolutely can incorporate domain-specific text — that's largely what it's for (teaching style, format, domain vocabulary). What it's bad at is *frequently changing facts*, not domain text in general.

**Why not D:** RAG adds retrieval latency and extra tokens (the retrieved context) to every call, which often makes it *more* expensive per request than a fine-tuned model with no extra context. Cost comparison depends on retrieval volume and context size, not a fixed rule.

**Interviewer's likely follow-up:** *"When would you reach for fine-tuning instead?"* (Answer: when you need to change behavior/format/style consistently rather than inject facts, when latency budget can't absorb retrieval, or when the "knowledge" is really a skill — like following a particular output schema — not a fact lookup.)
</details>

### Q4.2 · RAG vs long-context · [Applied]

A candidate says: "Context windows are huge now — 200K+ tokens — so RAG is basically obsolete, just stuff everything into the prompt." What's the strongest counter-argument?

- **A.** Long-context models can't process more than a few thousand tokens reliably
- **B.** Stuffing the full corpus into every call means paying to re-process (or re-cache) the same tokens on every request, and retrieval quality on "needle in haystack" tasks still degrades with volume even when the raw context window allows it
- **C.** Long-context prompts are not supported by any current LLM API
- **D.** RAG produces higher-quality embeddings than long-context models can process

<details>
<summary>Answer</summary>

**B**

**Why B:** Even with generous context windows, two things don't go away: cost/latency scale with tokens sent (whole-corpus-per-call is expensive unless heavily cached), and empirically, model attention over very long contexts is not uniform — relevant information in the middle of a huge context is retrieved less reliably than a well-targeted, retrieval-narrowed context. Long context is a tool that raises the ceiling on how much you *can* include, not a reason to skip narrowing down to what's *relevant*.

**Why not A:** This is factually wrong for modern models — context windows in the hundreds of thousands of tokens are common now, and this is the premise the candidate's claim (wrongly) leans on being false.

**Why not C:** Also factually wrong — long-context prompting is a standard, supported API pattern.

**Why not D:** Embeddings and long-context processing aren't in a quality competition with each other — they solve different problems (finding relevant chunks vs. processing whatever's put in front of the model). This option confuses two unrelated mechanisms.

**Interviewer's likely follow-up:** *"So when would you actually just stuff everything into context instead of doing retrieval?"* (Answer: small, bounded corpora that fit comfortably with room to spare; when the full corpus benefits from prompt caching across many requests; or during prototyping before investing in a retrieval pipeline.)
</details>

### Q4.3 · Chunk size and overlap · [Applied]

You're chunking a corpus of legal contracts using fixed 200-token chunks with no overlap. Retrieval quality is poor — the model frequently answers "I don't have enough information" even when the answer clearly exists in the source documents. What's the most likely fix to try first?

- **A.** Switch to a larger embedding model
- **B.** Add overlap between chunks and/or increase chunk size so that clause-level context (definitions, cross-references) isn't severed mid-thought at arbitrary token boundaries
- **C.** Lower the temperature on the generation call
- **D.** Increase the number of chunks retrieved (k) without changing chunk size

<details>
<summary>Answer</summary>

**B**

**Why B:** Legal contracts are full of cross-references ("as defined in Section 3.2") and clauses that only make sense with surrounding context. Fixed 200-token chunks with zero overlap will frequently cut a defined term away from its definition or split a clause mid-sentence, leaving no single chunk with a complete, self-contained thought. Increasing chunk size and/or adding overlap keeps more of that local context intact per chunk.

**Why not A:** A stronger embedding model improves semantic matching between query and chunk, but if the chunk itself doesn't contain the needed information (because it was severed), no embedding model fixes that.

**Why not C:** Temperature affects how the model phrases its response, not what information is available to it. This wouldn't touch an "insufficient information" failure.

**Why not D:** Retrieving more of the same badly-cut chunks doesn't reconstruct the severed context — you'd just be feeding the model more fragments, some of which may still individually lack the needed information. It could even hurt precision.

**Interviewer's likely follow-up:** *"What's the downside of just cranking chunk size way up to be safe?"* (Answer: larger chunks mean less precise retrieval — a matched chunk now carries more irrelevant filler alongside the relevant part, more tokens per chunk retrieved, higher cost, and you can crowd out other relevant chunks within your token budget.)
</details>

### Q4.4 · Semantic vs fixed chunking · [Recall]

What's the main practical advantage of semantic (or document-structure-aware) chunking over fixed-size chunking?

- **A.** It's always faster to compute than fixed-size chunking
- **B.** It produces chunks aligned to natural content boundaries (headings, paragraphs, list items), so each chunk is more likely to be a coherent, self-contained unit of meaning rather than an arbitrary token-count cut
- **C.** It eliminates the need for an embedding model
- **D.** It guarantees every chunk fits within the model's context window

<details>
<summary>Answer</summary>

**B**

**Why B:** Semantic/structure-aware chunking splits on meaningful boundaries — section headers, paragraph breaks, sentence groups with high internal similarity — rather than "every 200 tokens regardless of what's there." The result is chunks that are more likely to represent one complete idea, which improves both retrieval precision (the chunk clearly relates to one topic) and generation quality (the chunk is a self-contained, readable unit).

**Why not A:** Semantic chunking is typically *more* computationally expensive than fixed-size chunking — it often requires parsing document structure or computing sentence-level embeddings to find natural breakpoints, versus fixed-size chunking's simple token counting.

**Why not C:** Chunking (of any strategy) happens before embedding, not instead of it — you still need to embed each resulting chunk to make it searchable.

**Why not D:** Chunk size and context-window fit are unrelated to chunking *strategy* — you can produce oversized chunks with semantic chunking just as easily as with fixed-size chunking if you don't also cap chunk length.

**Interviewer's likely follow-up:** *"What's a case where semantic chunking would actually hurt you?"* (Answer: highly uniform, dense technical data like tables or logs, where there's no meaningful "semantic boundary" to find and the added complexity just introduces inconsistent, hard-to-predict chunk sizes for no real benefit.)
</details>

### Q4.5 · Cosine similarity vs Euclidean · [Recall]

Why is cosine similarity typically preferred over Euclidean distance for comparing text embeddings?

- **A.** Cosine similarity is always faster to compute than Euclidean distance
- **B.** Cosine similarity measures the angle between vectors, which captures directional (semantic) similarity independent of vector magnitude — and embedding magnitude often reflects factors like text length rather than meaning
- **C.** Euclidean distance can only be used with binary vectors
- **D.** Cosine similarity guarantees results between 0 and 1, while Euclidean distance does not

<details>
<summary>Answer</summary>

**B**

**Why B:** Two embeddings can point in nearly the same direction (same meaning) but have different magnitudes — often correlated with factors like input length or how "intense" certain features are, not semantic content. Cosine similarity normalizes this out by only measuring the angle between vectors, so it isolates "are these semantically pointing the same way" from "how long/large was the underlying text."

**Why not A:** Computational cost is roughly comparable (both are simple vector operations); this isn't the deciding factor.

**Why not C:** Euclidean distance works fine on any real-valued vectors, not just binary ones.

**Why not D:** This gets the framing backwards and is imprecise — cosine *similarity* is typically bounded in [-1, 1] (or [0,1] for non-negative embeddings), but that boundedness isn't really the reason it's preferred; the magnitude-independence is the actual reason, and stating a "guarantee" like this is the kind of half-true fact that sounds rigorous but isn't the real mechanism.

**Interviewer's likely follow-up:** *"Is there a case where you'd actually want Euclidean distance instead?"* (Answer: when magnitude itself is meaningful signal in your embedding space — some specialized or non-text embeddings encode intensity/scale as meaningful information, in which case normalizing it away with cosine would throw away real signal. For standard text embeddings, this is rare.)
</details>

### Q4.6 · Vector store selection · [Design]

A team is choosing a vector store for a new RAG pipeline. They need: metadata filtering by tenant ID on every query, tight integration with an existing PostgreSQL deployment (their ops team doesn't want a new database to operate), and moderate scale (a few million vectors). What should most heavily influence the choice here?

- **A.** Whichever vector store has the largest published benchmark numbers for raw query latency
- **B.** Operational fit — pgvector as a Postgres extension avoids introducing a new system to operate, and Postgres's existing row-level security/filtering can naturally support tenant-scoped metadata filtering, at a scale (millions, not billions) where a dedicated vector database's extra throughput isn't the bottleneck
- **C.** Whichever store is open source, regardless of other factors
- **D.** The store with the most embedding-model integrations pre-built

<details>
<summary>Answer</summary>

**B**

**Why B:** The stated constraints — no new system to operate, tenant-scoped filtering, moderate scale — point directly at pgvector's core value proposition: it's "just Postgres," so it inherits existing operational tooling, backup strategy, and access-control patterns the team already has, and at millions (not billions) of vectors it performs well enough that a dedicated vector database's superior raw throughput isn't the deciding factor. Purpose-built vector stores (Qdrant, Pinecone, Weaviate) earn their keep at higher scale or when you need vector-specific features Postgres doesn't have — not here.

**Why not A:** Raw benchmark latency numbers, taken alone, ignore the operational cost the team explicitly flagged as a constraint (not wanting a new system). Optimizing for a benchmark number that isn't the actual bottleneck is a classic mis-prioritization.

**Why not C:** Open-source-ness alone doesn't address any of the three stated constraints — it's a real factor sometimes, but treating it as the deciding one here ignores what was actually asked.

**Why not D:** Pre-built embedding-model integrations are a convenience, not a constraint that was raised. It's a reasonable secondary factor, not the primary driver given what the team said they cared about.

**Interviewer's likely follow-up:** *"At what point would you reconsider and move off pgvector?"* (Answer: when vector-specific scale — tens/hundreds of millions+ of vectors — or query throughput starts to genuinely strain Postgres, or when you need vector-database-native features like advanced hybrid search tooling, sharding, or multi-region replication purpose-built for vectors.)
</details>

### Q4.7 · Hybrid search · [Applied]

A RAG pipeline using pure dense (embedding) retrieval performs poorly on queries containing exact product SKUs like "XR-4471-B", even though those SKUs appear verbatim in the source documents. What's happening, and what's the fix?

- **A.** The embedding model wasn't trained on product data — retrain it on SKU examples
- **B.** Dense embeddings capture semantic meaning but are weak at exact lexical/token matching for rare, out-of-vocabulary-like strings such as SKUs — adding a sparse/keyword method (BM25) alongside dense search, combined via hybrid search, catches the exact-match case
- **C.** The chunk size is too large — reduce it
- **D.** Switch entirely from dense to sparse (BM25) retrieval

<details>
<summary>Answer</summary>

**B**

**Why B:** Dense embeddings are optimized to capture *meaning* — they're good at "these two sentences are about the same topic" even with different wording. But an alphanumeric SKU like "XR-4471-B" carries little semantic content the embedding model can leverage; two different SKUs might embed close together because they're both "product-code-shaped" strings, without the model distinguishing the specific characters. BM25 (or any sparse/lexical method) matches on the actual tokens, so it excels exactly where dense retrieval is weak: rare, exact strings. Hybrid search combines both, typically outperforming either alone.

**Why not A:** Retraining an embedding model is a heavyweight, expensive fix for a problem that hybrid search solves architecturally without any retraining.

**Why not C:** Chunk size affects how much context surrounds a match, not whether the retrieval method itself can find exact lexical matches. This isn't the actual mechanism at play.

**Why not D:** Going pure sparse would likely fix the SKU problem but at the cost of losing dense retrieval's strength on paraphrased, semantically-similar-but-differently-worded queries — you'd be trading one failure mode for another rather than getting the best of both.

**Interviewer's likely follow-up:** *"How do you combine dense and sparse scores in practice?"* (Answer: common approaches include reciprocal rank fusion (RRF) to combine rankings without needing to normalize incompatible score scales, or a weighted linear combination of normalized scores — RRF is more common because dense cosine similarity and BM25 scores aren't on comparable scales.)
</details>

### Q4.8 · Reranking · [Applied]

A pipeline retrieves the top 20 chunks by embedding similarity, then passes all 20 directly to the LLM for generation. Adding a reranking step that scores all 20 more precisely and passes only the top 5 to the LLM improves answer quality. Why would this help, given that the original 20 chunks presumably included the 5 most relevant ones already?

- **A.** Rerankers are always more accurate than embedding models, so they can find relevant chunks the embedding search missed entirely
- **B.** Rerankers use a more expensive, query-and-chunk-jointly-scored model (typically a cross-encoder) that's more precise at ranking than the cheaper bi-encoder used for initial retrieval — so even though the right chunks were likely already in the top 20, reranking surfaces the *most* relevant ones and cuts the irrelevant noise the LLM would otherwise have to sift through
- **C.** Reranking reduces the total number of tokens sent to the embedding model
- **D.** Reranking replaces the need for chunking entirely

<details>
<summary>Answer</summary>

**B**

**Why B:** Initial dense retrieval typically uses a bi-encoder — it embeds the query and each chunk independently, then compares vectors, which is fast enough to search over millions of chunks but less precise. A reranker (often a cross-encoder) jointly processes the query and each candidate chunk together, producing a much more accurate relevance score — but it's too expensive to run over the whole corpus, so it's only practical on a small candidate set (the initial top-k). The value isn't "finding chunks embedding search missed" — it's precisely re-ordering the candidates already found, and cutting the noisy long tail (chunks 6–20) that would otherwise dilute the LLM's attention with less-relevant context.

**Why not A:** Rerankers don't search the full corpus — they only re-score chunks that were already retrieved. If a genuinely relevant chunk never made it into the top 20 from initial retrieval, reranking can't recover it; that's a retrieval-stage problem, not something reranking fixes.

**Why not C:** Reranking happens after initial retrieval, scoring already-retrieved chunks against the query — it has nothing to do with the embedding model's token processing.

**Why not D:** Reranking and chunking are separate stages entirely; reranking operates on already-chunked, already-retrieved candidates.

**Interviewer's likely follow-up:** *"What's the cost tradeoff of adding a reranking stage?"* (Answer: extra latency and compute cost per query — you're running a heavier model, even if only on a small candidate set — so it's a tradeoff of added latency/cost against improved precision, usually worth it once k is large enough that noise is hurting generation quality.)
</details>

### Q4.9 · Metadata filtering · [Applied]

A RAG pipeline over company documents keeps surfacing outdated policy documents alongside current ones, even when both are semantically similar to the query, and the answer sometimes cites the outdated version. What's the most direct fix?

- **A.** Increase the embedding model's dimensionality
- **B.** Add a `status: current/archived` (or effective-date) metadata field to each document at ingestion, and filter retrieval to only search `current` (or apply a recency-aware filter) before ranking by similarity
- **C.** Lower the number of chunks retrieved (k)
- **D.** Add a system prompt instruction telling the model to "prefer newer information"

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a metadata problem, not a semantic-similarity problem — an outdated policy document can be highly semantically similar to a query (it's about the same topic) while being the *wrong* document to retrieve. Metadata filtering lets you exclude or deprioritize archived documents structurally, before similarity ranking even happens, which is a much more reliable fix than hoping similarity scores happen to favor the current version.

**Why not A:** Embedding dimensionality affects representational capacity for semantic nuance, not whether outdated documents are excluded — this doesn't address the actual failure.

**Why not C:** Reducing k just returns fewer results; if the outdated document ranks highly by similarity, a smaller k could make things worse (less room for a current document to also appear), not better.

**Why not D:** Instructing the model to "prefer newer information" in the prompt relies on the model correctly inferring recency from content it's given — unreliable and doesn't stop the outdated document from being retrieved and taking up context budget in the first place. It's a weak patch on top of a retrieval-stage problem.

**Interviewer's likely follow-up:** *"What happens if a document doesn't have clean metadata — say, it's an old scanned PDF with no structured fields?"* (Answer: you need an ingestion-time enrichment step — extract or infer metadata like date, source, status before indexing, possibly with an LLM pass over unstructured documents — because filtering can't operate on metadata that was never captured.)
</details>

### Q4.10 · Query rewriting and HyDE · [Applied]

A user asks a RAG chatbot: "why did it break again lol." Retrieval on this raw query returns poor results. What technique addresses this, and why?

- **A.** Increase chunk overlap to compensate for vague queries
- **B.** Query rewriting — using the LLM to reformulate the vague, conversational query into a more specific, retrieval-friendly form (potentially using conversation history for context) before embedding it and searching
- **C.** Switch the vector store to one with higher recall@k
- **D.** Ask the reranker to interpret informal language

<details>
<summary>Answer</summary>

**B**

**Why B:** "Why did it break again lol" is casual, context-dependent, and underspecified — it assumes shared context (what's "it"?) that a bare embedding search over a document corpus can't infer. Query rewriting uses the LLM (with access to conversation history) to turn this into something like "recurring failure cause for [specific system named earlier in conversation]" — a query whose embedding will actually land near relevant chunks. This directly targets the mismatch between how users write casually and how documents are written formally.

**Why not A:** Chunk overlap is a document-side, ingestion-time decision — it doesn't change anything about how a vague query gets embedded or matched. Wrong stage of the pipeline entirely.

**Why not C:** A vector store's recall@k capability doesn't help if the query embedding itself doesn't semantically resemble the relevant content — this is a "garbage in" problem at the query stage, not a retrieval-engine capability problem.

**Why not D:** Rerankers score candidate chunks against a query — they don't fix an underlying query that never retrieved the right candidates to rerank in the first place.

**Interviewer's likely follow-up:** *"What's HyDE and how is it different from plain query rewriting?"* (Answer: HyDE — Hypothetical Document Embeddings — has the LLM generate a *hypothetical answer* to the query first, then embeds that hypothetical answer and searches with it, on the theory that a plausible-looking answer will be semantically closer to real matching documents than the bare question is. It's a specific rewriting technique, not a different category.)
</details>

### Q4.11 · Retrieval failure vs generation failure · [Applied]

A RAG system gives a factually wrong answer to "What's our refund policy for enterprise customers?" You check the logs: the retrieved chunks *do* contain the correct enterprise refund policy. What kind of failure is this, and what should you check next?

- **A.** This is a retrieval failure — the embedding model needs retuning
- **B.** This is a generation failure — the correct information was retrieved but the model didn't use it correctly. Check the prompt template (is the retrieved context actually being included?), whether conflicting information (e.g. a general refund policy chunk) is also in context and confusing the model, and whether the model is hallucinating despite having the right grounding
- **C.** This is a chunking failure — increase chunk size
- **D.** This is an embedding dimensionality failure

<details>
<summary>Answer</summary>

**B**

**Why B:** The diagnostic split is simple and important: if the correct information was retrieved but the answer is still wrong, retrieval did its job — the failure is downstream, in generation. Common causes: the prompt template not actually inserting the retrieved context where you think it is, multiple retrieved chunks containing conflicting information (e.g. a general policy alongside the enterprise-specific one) that the model resolves incorrectly, or the model simply hallucinating/ignoring provided context. None of these are retrieval problems.

**Why not A:** Retuning the embedding model addresses whether the *right chunks get found* — but the logs already confirm the right chunk was found. This diagnosis targets the wrong stage.

**Why not C:** Chunk size affects what gets retrieved and how complete each chunk's information is — again, not relevant once you've confirmed the correct information was already in the retrieved context.

**Why not D:** Embedding dimensionality is a retrieval-stage lever with no bearing on what the model does with information once retrieved.

**Interviewer's likely follow-up:** *"How would you catch this systematically, not just by manually checking logs on one bad answer?"* (Answer: build an eval that separates retrieval metrics — recall@k, "was the gold chunk in the retrieved set" — from generation/answer-quality metrics like groundedness or faithfulness scoring, so a dashboard can immediately tell you which stage regressed rather than requiring manual log inspection every time.)
</details>

### Q4.12 · Chunking strategy · [Applied]

A support-docs RAG pipeline returns answers that are factually correct but
consistently miss information that spans section boundaries. Retrieval recall@10
is 0.91. What is the most likely cause?

- **A.** The embedding model is undertrained on domain vocabulary
- **B.** Chunk size is too small relative to the semantic units in the source
- **C.** The reranker is over-weighting lexical overlap
- **D.** Temperature is set too high on the generation call

<details>
<summary>Answer</summary>

**B**

**Why B:** High recall@10 means retrieval is finding the right *documents*. The
failure is that individual chunks don't contain complete thoughts — information
spanning a boundary gets split, so no single chunk carries the full answer.
Increasing chunk size or overlap, or moving to semantic/recursive chunking, addresses this.

**Why not A:** A weak embedding model would depress recall. Recall is 0.91.

**Why not C:** A reranking problem would also show up as lower effective recall on
the retrieved set, and wouldn't specifically correlate with boundary-spanning content.

**Why not D:** Temperature affects phrasing and creativity, not what information
is available to the model.

**Interviewer's likely follow-up:** *"Okay — so you increase chunk size. What
breaks?"* (Answer: precision drops, more irrelevant context per chunk, higher
token cost, and you may exceed the context budget once you retrieve k chunks.)
</details>

### Q4.13 · Retrieval metrics · [Recall]

Recall@10 is 0.95 but MRR (mean reciprocal rank) is 0.3 for a retrieval pipeline. What does this combination tell you?

- **A.** The pipeline is broken and finds almost nothing relevant
- **B.** The relevant chunk is almost always somewhere in the top 10, but it's rarely ranked first or second — it's often buried lower in the list, which hurts if the LLM only reliably attends to the first few results or if k is set lower than 10 downstream
- **C.** Recall@10 and MRR measure the same thing, so this combination is impossible
- **D.** The embedding model has too many dimensions

<details>
<summary>Answer</summary>

**B**

**Why B:** Recall@10 = 0.95 means the correct chunk is present within the top 10 results 95% of the time — retrieval is finding it. MRR = 0.3 means, on average, the correct chunk's rank position is low-ish (roughly around position 3, since MRR ≈ 1/average-rank) — it's present but not consistently near the top. This is a genuinely useful and common combination to see: "coverage is good, ranking precision needs work" — which points you toward reranking rather than toward retrieval recall improvements.

**Why not A:** A recall@10 of 0.95 is strong — this directly contradicts "finds almost nothing relevant."

**Why not C:** These are different metrics measuring different things (presence-within-top-k vs. rank-position-of-first-relevant-result) and frequently diverge exactly like this — that's the point of tracking both.

**Why not D:** Embedding dimensionality isn't diagnosable from this metric pair at all — this option doesn't engage with what the numbers actually mean.

**Interviewer's likely follow-up:** *"Given these two numbers, what would you fix first?"* (Answer: add or improve reranking — the retrieval stage is already finding the right chunk, so the fix is in ranking precision, not in the retrieval/embedding stage. Improving recall further wouldn't move MRR much.)
</details>

### Q4.14 · Handling updating documents · [Design]

You're designing ingestion for a RAG pipeline over a knowledge base where articles get edited multiple times per day. What's the most important design decision to get right?

- **A.** Using the largest available embedding model for maximum semantic accuracy
- **B.** A re-indexing strategy that reliably detects changed documents (via webhook, polling with content hashes, or a CDC-style trigger) and atomically replaces the old chunks/embeddings for that document, so stale chunks never coexist with updated ones in the index
- **C.** Choosing a vector store with the lowest query latency
- **D.** Maximizing chunk overlap to reduce the chance of missing content

<details>
<summary>Answer</summary>

**B**

**Why B:** The core risk with frequently-updated content is staleness and, worse, *inconsistency* — old and new chunks of the same document coexisting in the index, so a query might retrieve an outdated fact right alongside its correction. A robust re-indexing pipeline needs (1) reliable change detection so edits actually trigger re-embedding, and (2) atomic replacement so you never serve a half-updated document — old chunks for an article should be fully retired the moment new ones are indexed, not left to linger.

**Why not A:** Embedding model size affects semantic match quality, not freshness — a bigger model on stale data still returns stale, wrong answers.

**Why not C:** Query latency is a real concern but secondary to correctness here — a fast query against a stale or inconsistent index is fast *and wrong*, which is worse than the update-freshness problem being asked about.

**Why not D:** Chunk overlap addresses information being severed across chunk boundaries — unrelated to whether an index reflects the current version of a document.

**Interviewer's likely follow-up:** *"What if re-indexing lags by a few minutes — is that acceptable?"* (Answer: usually yes for most use cases — a few minutes of eventual consistency is a reasonable tradeoff against re-indexing cost, as long as it's a bounded, known lag rather than an unbounded or silent one, and as long as nothing downstream assumes real-time freshness without checking.)
</details>

### Q4.15 · Multi-tenant retrieval · [Design]

You're building RAG for a B2B SaaS product where each customer's documents must never be retrievable by another customer, even accidentally. What's the most robust way to enforce this?

- **A.** Trust the LLM's system prompt instruction: "only discuss documents belonging to this customer"
- **B.** Enforce tenant isolation at the retrieval/query layer itself — via separate indexes per tenant, or a mandatory tenant-ID metadata filter applied before similarity search runs, so cross-tenant documents are structurally never candidates, not just discouraged from being cited
- **C.** Rely on the reranker to deprioritize other tenants' documents
- **D.** Ask customers to use different embedding models per tenant

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a security boundary, not a quality-of-answer concern — it needs to be enforced structurally, before retrieval even happens, not hoped for via model behavior. Separate per-tenant indexes (strongest isolation) or a hard, mandatory tenant-ID filter applied at the database/query layer (so a document from another tenant is never even a retrieval candidate) both make cross-tenant leakage structurally impossible rather than merely unlikely.

**Why not A:** System prompt instructions are a request to the model, not an enforcement mechanism — the model can still be manipulated (prompt injection, edge cases, plain model error) into referencing content it was told not to. This is the trust-boundary mistake covered more deeply in file 07, and it's a serious one to make on data isolation.

**Why not C:** Reranking operates on documents that were *already retrieved* — if a cross-tenant document made it into the candidate set, the leak already happened in terms of context exposure; deprioritizing its rank doesn't undo that.

**Why not D:** Using different embedding models per tenant is operationally absurd, doesn't actually enforce isolation (you'd still need a filter to prevent cross-index search), and doesn't address the actual problem.

**Interviewer's likely follow-up:** *"How would you catch a tenant-isolation bug before it hits production?"* (Answer: an automated test suite that seeds documents for two fake tenants and asserts that queries under tenant A's context never surface tenant B's chunks — run this in CI on every change to the retrieval layer, not as a manual one-off check.)
</details>

### Q4.16 · Chunk size / precision tradeoff · [Applied]

After increasing chunk size from 200 to 800 tokens to fix boundary-spanning information loss (as in Q4.12), a team notices answer quality actually drops for simple, narrow factual questions. Why?

- **A.** Larger chunks always produce worse embeddings regardless of query type
- **B.** Larger chunks each cover more topics/content, so for a narrow question, a matched chunk now carries proportionally more irrelevant surrounding text — diluting the signal the model has to work with and potentially crowding out other relevant chunks within the token budget at a fixed k
- **C.** The vector store can't handle chunks larger than 200 tokens
- **D.** Increasing chunk size always breaks metadata filtering

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the classic precision/recall tradeoff of chunk size, playing out in reverse from Q4.12: bigger chunks reduce the *split-thought* problem but increase the *noise-per-chunk* problem. For a narrow factual question, a large chunk mostly filled with unrelated context makes it harder for the model to zero in on the actual answer, and at a fixed retrieval k, you're now spending more of your token budget on filler within each chunk, potentially pushing out other genuinely relevant chunks that would've fit at the smaller size.

**Why not A:** Chunk size doesn't uniformly degrade embedding quality — it changes what's *in* the chunk being embedded, with different tradeoffs for different query types, not a blanket quality hit.

**Why not C:** Vector stores don't have a meaningful hard limit around 200 vs 800 tokens for typical chunk sizes — this isn't a real technical constraint at this scale.

**Why not D:** Metadata filtering operates independently of chunk content size — this option invents an unrelated failure mode.

**Interviewer's likely follow-up:** *"How would you get the benefit of both without picking one fixed size?"* (Answer: hierarchical/multi-granularity retrieval — index both small and large chunks, or index small chunks but retrieve their surrounding larger parent section for context — sometimes called "small-to-big" retrieval; alternatively, tune chunk size per document type based on its typical content density.)
</details>

### Q4.17 · Recall@k vs precision@k · [Recall]

What's the difference between recall@k and precision@k in retrieval evaluation?

- **A.** They're two names for the same metric
- **B.** Recall@k measures what fraction of *all* relevant items were found within the top k results; precision@k measures what fraction of the top k results *retrieved* were actually relevant
- **C.** Recall@k only applies to sparse retrieval, precision@k only to dense retrieval
- **D.** Precision@k is always higher than recall@k for a given k

<details>
<summary>Answer</summary>

**B**

**Why B:** Recall@k answers "of everything relevant that exists, how much did we capture in our top k?" — it's about coverage. Precision@k answers "of what we returned in the top k, how much of it was actually relevant?" — it's about purity of the result set. A pipeline can have high recall (found everything relevant, somewhere in a big k) with low precision (buried among a lot of irrelevant noise), or vice versa — they capture genuinely different failure modes.

**Why not A:** They measure different things, as above — conflating them would make it impossible to diagnose whether a retrieval problem is "we're missing relevant content" (recall) or "we're returning too much junk" (precision).

**Why not C:** Both metrics apply equally to any retrieval method — dense, sparse, or hybrid. This isn't a real constraint.

**Why not D:** There's no general rule that one is always higher — the relationship depends on k, the size of the relevant set, and retrieval quality; for a small relevant set and large k, recall tends to be higher, but that's situational, not a rule.

**Interviewer's likely follow-up:** *"If you could only optimize for one of these two, which would you pick for a RAG system and why?"* (Answer: usually recall matters more at the retrieval stage, since reranking downstream can improve precision on an already-adequate candidate set — but if the relevant chunk was never retrieved at all, no amount of reranking recovers it. So bias toward recall in initial retrieval, precision in what you finally pass to generation.)
</details>

### Q4.18 · NDCG · [Recall]

What does NDCG (Normalized Discounted Cumulative Gain) add on top of a metric like recall@k?

- **A.** NDCG ignores ranking position entirely and only checks presence in the result set
- **B.** NDCG accounts for both relevance *grade* (not just binary relevant/irrelevant, but how relevant) and rank position (a highly relevant result near the top contributes more than the same result buried lower), normalized against the ideal possible ranking
- **C.** NDCG is only usable for exactly k=10
- **D.** NDCG replaces the need for a labeled evaluation set

<details>
<summary>Answer</summary>

**B**

**Why B:** Recall@k treats relevance as binary and position-agnostic within the top k. NDCG is richer on two axes: it supports graded relevance (a chunk can be "somewhat relevant" vs "exactly the answer," not just yes/no), and it discounts a result's contribution based on where it ranks — a perfect match at position 1 is worth more than the same match at position 8. It then normalizes against the best-possible ordering (Ideal DCG) so scores are comparable across queries with different numbers of relevant items.

**Why not A:** This describes something closer to plain recall, and directly inverts NDCG's actual purpose — position sensitivity is the whole point of the "discounted" part of the name.

**Why not C:** NDCG@k works for any chosen k; there's no special restriction to 10 — that number just shows up commonly in examples and benchmarks.

**Why not D:** NDCG still requires ground-truth relevance judgments (ideally graded ones) to compute against — it doesn't remove the need for a labeled eval set, it just makes better use of richer labels if you have them.

**Interviewer's likely follow-up:** *"When would you bother with graded relevance labels instead of just binary relevant/not-relevant?"* (Answer: when "close but not quite the best chunk" is a meaningfully different outcome from "completely irrelevant" for your use case — e.g. legal or medical retrieval, where ranking the *most* authoritative source first matters, not just getting something roughly on-topic into the top k.)
</details>

### Q4.19 · Embedding dimensionality · [Recall]

A team is choosing between a 384-dimension and a 1536-dimension embedding model for a RAG pipeline. What's the main tradeoff?

- **A.** Higher dimensionality always produces strictly better retrieval quality with no downsides
- **B.** Higher dimensionality can capture more semantic nuance but costs more to store and search (index size, memory, query latency scale with dimension count), and past a point, the quality gain often plateaus for a given domain
- **C.** Dimensionality only affects the embedding model's training time, not inference
- **D.** Lower-dimension embeddings can't be used with cosine similarity

<details>
<summary>Answer</summary>

**B**

**Why B:** More dimensions give the model more "room" to represent fine-grained semantic distinctions, which can help quality — but every dimension adds storage cost (index size scales roughly linearly with dimension count) and compute cost for similarity search at scale. In practice, quality gains from more dimensions diminish past a certain point for a given domain and corpus size, so the real decision is a cost/quality tradeoff, evaluated empirically on your own data — not "bigger is strictly better."

**Why not A:** This is the trap the question is testing — there's no free lunch; storage and latency costs are real and scale with dimension count, and quality gains aren't guaranteed to justify them for every use case.

**Why not C:** Dimensionality affects inference cost (every embedding call and every similarity computation) just as much as training — this option is simply wrong.

**Why not D:** Cosine similarity works over vectors of any dimensionality — there's no restriction tying it to dimension count.

**Interviewer's likely follow-up:** *"How would you actually decide between the two for a specific project?"* (Answer: run both on a representative eval set with your own labeled queries and measure recall@k/NDCG, then weigh the quality delta against the storage/latency cost delta at your expected scale — don't decide from published benchmarks alone, since domain fit varies.)
</details>

### Q4.20 · Groundedness / hallucination in RAG · [Applied]

A RAG-based answer sounds confident and well-formed, and cites "according to the retrieved policy document." When you check, no such document was actually retrieved for that query — the model invented the citation. What's this failure mode, and what's a direct mitigation?

- **A.** This is a retrieval failure — improve recall@k
- **B.** This is a groundedness/hallucination failure — the model generated a plausible-sounding claim not supported by the actual retrieved context. Mitigate with a groundedness check step (verify claims against retrieved chunks, e.g. via a faithfulness-scoring pass or requiring inline citations that are then validated) and/or a stricter prompt instructing the model to answer only from provided context and say so explicitly when it can't
- **C.** Switch to a larger embedding model
- **D.** This is expected LLM behavior and can't be meaningfully reduced

<details>
<summary>Answer</summary>

**B**

**Why B:** The model fabricating a citation to a document that was never in its context window is a pure generation-side hallucination — it has nothing to do with what was or wasn't retrieved (retrieval never even offered that document). Mitigations operate at the generation/validation stage: a post-generation faithfulness check that verifies each claim is actually supported by the retrieved chunks, prompt-level instructions that explicitly permit "I don't have that information" instead of pressuring the model to always produce a confident answer, and/or citation validation that programmatically checks cited sources actually appear in the retrieved set.

**Why not A:** Recall@k measures whether relevant documents were retrieved — it has no bearing on the model inventing a citation to something that was never retrieved in the first place. This is a downstream failure, matching the retrieval-vs-generation distinction from Q4.11.

**Why not C:** Embedding model quality affects what gets retrieved, not whether the generation step fabricates unsupported claims about what it received.

**Why not D:** While hallucination can't be reduced to zero, treating it as unmitigable is wrong and is exactly the kind of answer that signals a candidate hasn't thought about production RAG — faithfulness scoring, stricter prompting, and citation validation meaningfully reduce (not eliminate) this failure mode.

**Interviewer's likely follow-up:** *"How would you measure groundedness at scale, not just spot-check individual answers?"* (Answer: an automated groundedness/faithfulness metric — often an LLM-as-judge pass that checks whether each claim in the generated answer is entailed by the retrieved context — run as part of a regular eval suite, covered more in file 06.)
</details>

### Q4.21 · Chunking document structure · [Design]

You're chunking a corpus that's a mix of prose documentation, code snippets, and tables. What's the strongest chunking approach?

- **A.** Apply one uniform fixed-token-size chunker across all content types for consistency
- **B.** Use content-type-aware chunking — treat code blocks and tables as atomic units (don't split them mid-structure), and use different strategies for prose (paragraph/semantic boundaries) vs. structured content (keep tables whole or split by row-group with headers repeated)
- **C.** Strip out all code and tables before chunking, since LLMs can't reason about them
- **D.** Convert everything to plain prose using an LLM before chunking

<details>
<summary>Answer</summary>

**B**

**Why B:** Different content types have fundamentally different "units of meaning." A code snippet split mid-function is often useless; a table split mid-row loses its header context and becomes uninterpretable numbers. Content-type-aware chunking respects the actual structure — treating code blocks as atomic (don't split inside them, chunk around them), and for tables, either keeping them whole (if small) or splitting by row groups while repeating the header row so each chunk stays self-interpretable.

**Why not A:** A uniform fixed-size chunker applied blindly is exactly what breaks code and tables — it has no awareness that "line 47 of this function" is meaningless without the rest of the function, unlike prose where a rough token cut is more forgiving.

**Why not C:** Stripping code and tables throws away real, often highly valuable information (LLMs are quite capable of reasoning over code and simple tables when given as intact context) — this loses information rather than solving the chunking problem.

**Why not D:** Converting code/tables to prose via an LLM adds cost, latency, and a real risk of introducing errors or losing precision (e.g., exact table values) in the conversion step, for no clear benefit over just handling structure natively.

**Interviewer's likely follow-up:** *"How do you decide chunk boundaries for a table with hundreds of rows?"* (Answer: chunk by logical row groups sized to fit comfortably in context, repeat column headers in every chunk so each one is independently interpretable, and consider whether the table would be better served by a different retrieval mechanism entirely — e.g. a SQL query against structured data rather than embedding-based RAG.)
</details>

### Q4.22 · When RAG is the wrong tool · [Design]

A team wants RAG to let their support bot compute a customer's current account balance by "retrieving" transaction records. What's wrong with this approach?

- **A.** Nothing — RAG can retrieve any type of document including transaction records
- **B.** A balance is a computed, exact, currently-true value, not a static fact to retrieve semantically — this calls for a direct database/API query (a tool call) that fetches and computes the real-time value, not embedding-based retrieval over a snapshot of transaction text, which risks staleness, incompleteness, and semantic-similarity-based retrieval missing exact records
- **C.** RAG only works for text under 500 tokens
- **D.** This requires fine-tuning, not RAG or tool use

<details>
<summary>Answer</summary>

**B**

**Why B:** RAG is built for retrieving *relevant unstructured content* by semantic similarity — it's a poor fit for values that need to be exact, current, and computed (like summing transactions). Embedding-based retrieval over transaction records risks missing some records (similarity ranking isn't guaranteed to surface every relevant transaction) and can't perform arithmetic. The right architecture is a tool call to a database/API that deterministically fetches and computes the actual balance — this is a "when NOT to use RAG" case, closely related to "when NOT to use an agent" reasoning in file 05.

**Why not A:** Technically you *could* retrieve transaction text, but "can" isn't "should" — it's the wrong mechanism for a task requiring completeness and exact computation, which is the point of this question.

**Why not C:** There's no such token-length restriction on RAG in general — this invents a constraint that isn't real.

**Why not D:** Fine-tuning doesn't solve this either — it still can't reliably perform exact, current arithmetic over live data; the right fix is a tool/function call to structured data, not a training-based approach at all.

**Interviewer's likely follow-up:** *"So when would retrieval over transaction data actually make sense?"* (Answer: for fuzzy, exploratory queries like "have I been charged something like this before" or "find transactions related to a merchant dispute" — semantic search over descriptions is genuinely useful there, versus "what's my exact current balance," which needs a deterministic computation.)
</details>

### Q4.23 · BM25 mechanics · [Recall]

What does BM25 fundamentally score, at a high level?

- **A.** Semantic similarity between the meanings of query and document
- **B.** A term-frequency-based relevance score that rewards documents where query terms appear frequently, while adjusting for document length and how rare/common each term is across the whole corpus (so common words contribute less than rare, distinctive ones)
- **C.** The cosine distance between the query embedding and document embedding
- **D.** How recently the document was indexed

<details>
<summary>Answer</summary>

**B**

**Why B:** BM25 is a lexical/keyword scoring function — it's essentially a refined TF-IDF. It scores how relevant a document is based on how often query terms actually appear in it (term frequency), while dampening the effect of very long documents (length normalization) and weighting rarer, more distinctive terms more heavily than common ones (inverse document frequency-like weighting). It has no concept of "meaning" — it's purely about matching actual tokens.

**Why not A:** Semantic similarity is what *dense/embedding* retrieval measures — BM25 is the sparse/lexical counterpart precisely because it does not attempt to capture meaning, only term overlap.

**Why not C:** Cosine distance between embeddings is a dense-retrieval concept entirely separate from BM25's term-frequency-based scoring.

**Why not D:** BM25 has no concept of recency or indexing time built into its scoring — that would need to be added as a separate signal (e.g., metadata filtering or a recency boost) on top of BM25 if desired.

**Interviewer's likely follow-up:** *"Why does BM25 still hold up so well against dense retrieval on some tasks, decades after it was introduced?"* (Answer: for queries that hinge on exact terms — names, codes, rare technical vocabulary — lexical matching is simply the right tool; dense embeddings can actually underperform there because they optimize for semantic generalization, which can blur exact-match precision. That complementary strength is exactly why hybrid search outperforms either alone.)
</details>

### Q4.24 · Cost of over-retrieving · [Applied]

A team sets k=50 for retrieval "to be safe" and passes all 50 chunks to the LLM on every query. What are the concrete costs of this choice?

- **A.** None — more context always improves answer quality
- **B.** Higher per-request token cost and latency (you're paying for and processing 50 chunks worth of tokens every time), increased risk of irrelevant/conflicting information diluting or confusing generation, and a higher chance of exceeding context budget when combined with conversation history and system prompt
- **C.** Vector stores cap retrieval at k=10, so this configuration wouldn't actually run
- **D.** Only the embedding cost increases; generation cost is unaffected by k

<details>
<summary>Answer</summary>

**B**

**Why B:** Every retrieved chunk becomes prompt tokens the LLM has to process, so higher k directly increases cost and latency (this is exactly the kind of tradeoff covered with real numbers in file 02). Beyond cost, more chunks means more opportunity for irrelevant or even contradictory information to end up in context, which can measurably hurt answer quality rather than help it (the "more context is always better" intuition is false past a point) — plus, at 50 chunks, it's easy to blow past your total context budget once you add system prompt and conversation history on top.

**Why not A:** This is the exact misconception the question is testing — more retrieved context is not free and is not unconditionally beneficial; irrelevant context actively competes for the model's attention and costs real money and latency.

**Why not C:** Vector stores don't impose a universal k=10 cap — k is a configurable parameter, and this option invents a constraint that doesn't generally exist.

**Why not D:** Generation cost scales with total input tokens, which includes all retrieved chunks — claiming only embedding cost is affected ignores the (usually larger) generation-side cost of processing 50 chunks on every call.

**Interviewer's likely follow-up:** *"How would you pick a better k than 'as high as possible to be safe'?"* (Answer: empirically, using your eval set — plot recall@k and answer-quality metrics against increasing k and find the point of diminishing returns, then weigh that against the cost/latency curve to pick a k that's good enough without paying for context that isn't helping.)
</details>

### Q4.25 · Embedding model domain fit · [Applied]

A general-purpose embedding model performs noticeably worse on a corpus of internal engineering runbooks full of company-specific acronyms and system names than it does on general web content. What's the most direct way to address this without a full custom-training project?

- **A.** Switch to a larger general-purpose embedding model — bigger models always generalize better to any domain
- **B.** Consider domain adaptation options short of full retraining: expanding the corpus with definitions/glossary chunks so acronyms have retrievable context, evaluating a few candidate embedding models against your own labeled queries rather than assuming general benchmarks transfer, and/or combining with BM25 (which handles exact acronym matches regardless of embedding quality) via hybrid search
- **C.** There's no fix short of fully fine-tuning a custom embedding model
- **D.** Increase chunk size until acronyms are always paired with their full definitions in every chunk

<details>
<summary>Answer</summary>

**B**

**Why B:** Before jumping to a full custom-training project, there are several lighter-weight levers: seeding the corpus with a glossary/definitions section so company-specific terms have retrievable context to anchor to; empirically comparing a few embedding models on your *actual* domain queries rather than trusting general-purpose leaderboards, since domain fit varies a lot and isn't predictable from generic benchmarks; and leaning on hybrid search — BM25 doesn't care about semantic domain fit at all, it just matches the acronym's literal tokens, so it's a strong complement exactly where dense embeddings struggle with jargon.

**Why not A:** Bigger general-purpose models don't reliably fix domain-specific vocabulary gaps — model size and domain fit are different axes; a bigger model trained on the same general web-scale data can still lack good representations for niche internal acronyms.

**Why not C:** This overstates the situation — full fine-tuning is one option, but it's usually not the first thing to try given its cost, when cheaper interventions (glossary context, model comparison, hybrid search) often close much of the gap.

**Why not D:** Guaranteeing every chunk pairs every acronym with its full definition isn't practical or reliable to enforce at ingestion time across an entire runbook corpus, and doesn't address queries that use the acronym without expecting a definition alongside it.

**Interviewer's likely follow-up:** *"If none of the lighter-weight fixes are enough, how would you scope a custom embedding fine-tuning project?"* (Answer: you'd need labeled query-document relevance pairs from your actual domain, likely gathered from real usage or SME-generated examples, then fine-tune or use techniques like contrastive learning on top of a strong base model — a meaningfully bigger investment, justified only once cheaper options are exhausted and the gap still matters.)
</details>

### Q4.26 · Retrieval latency budget · [Applied]

A RAG pipeline's total response time is dominated by the retrieval step (800ms) rather than generation (400ms), and the product needs sub-1-second responses. What are plausible causes to investigate first?

- **A.** The LLM's temperature setting is too high
- **B.** Likely culprits: an unoptimized vector index (e.g., exact nearest-neighbor search instead of an approximate index like HNSW), a reranking step adding a heavy extra model call, network round-trips to a remote vector database, or too high a k value being searched over
- **C.** Retrieval latency is fixed and can't be reduced below the embedding model's inherent inference time
- **D.** Switching from cosine similarity to Euclidean distance will fix latency

<details>
<summary>Answer</summary>

**B**

**Why B:** 800ms for retrieval alone is unusually slow and worth investigating across several real, common causes: whether the vector index uses approximate nearest-neighbor search (HNSW, IVF) versus slower exact search; whether a reranking cross-encoder is adding meaningful latency (rerankers are notably heavier than initial retrieval); network latency if the vector database is a remote, separately-hosted service; and whether k or the corpus size is large enough to matter at the current index configuration.

**Why not A:** Temperature is a generation-time sampling parameter — it has zero effect on retrieval latency, which is a separate stage entirely.

**Why not C:** Retrieval latency is very much reducible through index choice, infrastructure, and configuration — treating it as a fixed floor ignores the actual levers available (this is close to a "trivia trap" answer that sounds authoritative but is wrong).

**Why not D:** Distance metric choice (cosine vs Euclidean) is not a meaningful latency lever for well-implemented vector search — both are cheap vector operations; this doesn't address any of the actual likely bottlenecks.

**Interviewer's likely follow-up:** *"What's the tradeoff of switching to an approximate nearest-neighbor index?"* (Answer: approximate methods trade a small amount of retrieval accuracy — you might occasionally miss the exact nearest neighbor — for a large latency/throughput improvement at scale; for most RAG use cases this tradeoff is well worth it since retrieval doesn't need to be provably exact.)
</details>

### Q4.27 · Query rewriting risks · [Design]

A team adds an LLM-based query rewriting step before retrieval. What's a real risk this introduces that didn't exist with raw-query retrieval?

- **A.** Query rewriting always improves retrieval, so there's no meaningful downside
- **B.** The rewriting step adds latency and cost (an extra LLM call before retrieval even starts), and if the rewrite mischaracterizes the user's actual intent, it can retrieve confidently-wrong results that are further from the original intent than the raw query would have found
- **C.** Query rewriting removes the need for an embedding model
- **D.** Query rewriting can only be used with sparse retrieval, not dense

<details>
<summary>Answer</summary>

**B**

**Why B:** Query rewriting isn't free — it's an extra LLM call in the critical path, adding latency and cost before retrieval even begins (compounding with the cost/latency arithmetic covered in file 02). And it introduces a new failure surface: if the rewrite step misinterprets what the user actually meant (which can happen, especially on ambiguous or terse queries), you now retrieve confidently against the *wrong* reformulated query — a failure mode that's arguably worse than a merely-imperfect raw-query search, because it's less obviously "the query was vague," and instead silently drifts from user intent.

**Why not A:** This is the misconception being tested — query rewriting is a real, measurable improvement in many cases but isn't unconditionally good, and treating it as risk-free misses a genuine production concern.

**Why not C:** Query rewriting operates upstream of retrieval — you still embed the rewritten query and search as normal; it doesn't replace the embedding model or the retrieval mechanism at all.

**Why not D:** Query rewriting is retrieval-method-agnostic — the rewritten query gets used for whatever retrieval approach is in place, dense, sparse, or hybrid.

**Interviewer's likely follow-up:** *"How would you catch a bad rewrite before it causes a bad answer?"* (Answer: log both the original and rewritten query for every request so you can audit divergence; optionally run retrieval on both and compare/merge results; and include rewrite-quality checks in your eval suite rather than trusting it silently.)
</details>

### Q4.28 · Chunk overlap tradeoff · [Recall]

What's the direct cost of adding overlap between chunks (e.g., 50-token overlap on 200-token chunks)?

- **A.** No cost — overlap is strictly beneficial with no downside
- **B.** Redundant storage and indexing of the overlapping content across multiple chunks, and a higher chance that near-duplicate chunks both get retrieved for the same query, taking up two slots in the top-k with substantially the same information instead of two genuinely different relevant pieces
- **C.** Overlap makes cosine similarity calculations invalid
- **D.** Overlap can only be used with fixed-size chunking, not semantic chunking

<details>
<summary>Answer</summary>

**B**

**Why B:** Overlap exists to prevent information from being fully severed at a chunk boundary — genuinely useful — but it isn't free: the overlapping text gets stored and embedded twice (or more), increasing index size and embedding cost proportionally, and at retrieval time, two overlapping chunks covering nearly the same content can both rank highly for the same query, effectively wasting two of your k retrieval slots on redundant information rather than surfacing two distinct relevant pieces.

**Why not A:** Treating overlap as costless ignores real storage/embedding cost and the redundant-retrieval risk — there's a genuine tradeoff to tune (how much overlap is worth the redundancy), not a free win.

**Why not C:** Overlap has no bearing on the validity of cosine similarity as a comparison metric — those are unrelated concepts.

**Why not D:** Overlap can be applied conceptually with semantic chunking too (e.g., including a bit of the previous/next semantic unit at chunk edges) — it's not exclusive to fixed-size chunking, though it's more mechanically straightforward to define for fixed-size chunks.

**Interviewer's likely follow-up:** *"How much overlap is typically reasonable?"* (Answer: there's no universal number — it's commonly 10-20% of chunk size as a starting point, but the right value depends on how information-dense and boundary-crossing the content is; it should be tuned against retrieval quality on your own eval set, not set by convention alone.)
</details>

### Q4.29 · Document-structure-aware chunking edge case · [Applied]

A markdown-based knowledge base has some pages that are a single 3,000-word paragraph with no headings, and others that are well-structured with many short subsections. A document-structure-aware chunker handles the structured pages well but produces one giant unchunked block for the unstructured page. What should happen?

- **A.** Nothing — leave the giant block as-is since it's technically one "structural unit"
- **B.** Fall back to a secondary strategy (fixed-size or sentence/paragraph-based chunking) when structural signals are absent or a structural unit exceeds a size threshold, so no single chunk becomes unreasonably large regardless of the source document's formatting quality
- **C.** Reject the unstructured document from ingestion entirely
- **D.** Manually add headings to every document before ingestion

<details>
<summary>Answer</summary>

**B**

**Why B:** Pure structure-aware chunking is only as good as the structure actually present in the source — real-world corpora are inconsistent, and a chunker needs a fallback for when structural signals (headings, clear paragraph breaks) are missing or a "unit" balloons past a reasonable size. A robust pipeline defines a size ceiling and falls back to sentence- or paragraph-based splitting within an oversized block, so quality degrades gracefully instead of producing a single giant, low-precision chunk.

**Why not A:** Leaving a 3,000-word block as one chunk defeats the purpose of chunking entirely — it'll retrieve as an all-or-nothing block with poor precision and likely exceed reasonable per-chunk token budgets.

**Why not C:** Rejecting real documents from ingestion because they're poorly formatted throws away legitimate content — the fix belongs in the chunking pipeline's robustness, not in gatekeeping the corpus.

**Why not D:** Manually reformatting every source document doesn't scale and isn't a pipeline design decision — it's an unsustainable manual workaround for what should be handled programmatically.

**Interviewer's likely follow-up:** *"How would you decide the size threshold that triggers the fallback?"* (Answer: empirically, based on your chunk-size sweet spot from eval testing — e.g., if your target chunk size is ~300-500 tokens, trigger fallback splitting on any structural unit exceeding maybe 2-3x that, tuned against actual retrieval quality rather than picked arbitrarily.)
</details>

### Q4.30 · Structured output from RAG · [Applied]

A RAG pipeline needs to extract a specific field (e.g., "contract end date") from retrieved contract chunks and return it in a strict JSON schema for downstream automation. What's the right way to combine RAG with this requirement?

- **A.** RAG and structured output are incompatible — you have to choose one or the other
- **B.** Retrieve the relevant chunk(s) as normal, then use the model's structured-output/JSON-mode capability on the generation call so the final response is schema-constrained, while the retrieved context still grounds the actual extracted value
- **C.** Structured output must come before retrieval, not after
- **D.** Use a regex over the retrieved chunk instead of an LLM call

<details>
<summary>Answer</summary>

**B**

**Why B:** RAG and structured output operate at different stages and compose naturally: retrieval finds the relevant grounding content (the contract chunk mentioning the end date), and the generation call — now given that context — is constrained via JSON mode/structured output (covered mechanically in file 02) to return the extracted value in the required schema. Combining them is standard practice, not a conflict.

**Why not A:** This is simply false — retrieval and generation-time output formatting are orthogonal concerns; nothing about RAG prevents constraining the generation call's output format.

**Why not C:** Structured output constrains what the *generation* call produces — it doesn't make sense "before retrieval," since there's nothing to extract yet at that point in the pipeline.

**Why not D:** A regex is brittle against natural-language variation in how dates/terms are phrased across different contracts, and doesn't leverage the LLM's ability to correctly identify the right field even when phrasing varies — a reasonable fallback for very rigid formats, but not the general answer for varied real-world contract text.

**Interviewer's likely follow-up:** *"What if the retrieved chunk doesn't actually contain the requested field?"* (Answer: the schema should allow for a null/not-found value rather than forcing the model to fabricate one to satisfy the schema — this is a place where over-constraining without an escape hatch actively encourages hallucination.)
</details>

### Q4.31 · Access control granularity · [Design]

Beyond simple tenant-level isolation (Q4.15), a document management RAG system needs *document-level* permissions — some users within the same tenant can see certain documents, others can't. What's the strongest approach?

- **A.** Filter results after retrieval, removing documents the user can't see from the returned list before passing to the LLM
- **B.** Apply the permission filter as part of the retrieval query itself (metadata filtering on allowed document/ACL IDs before or during similarity search), so restricted documents are never retrieved as candidates in the first place — not filtered out afterward, which still exposes them to the retrieval/ranking process and risks leaking existence or partial content through timing, ranking, or logging side channels
- **C.** Give every user access to every document since document-level permissions are too complex for RAG
- **D.** Encrypt all documents so only permission checks matter, not retrieval-time filtering

<details>
<summary>Answer</summary>

**B**

**Why B:** This extends the same principle from Q4.15 (enforce isolation structurally, at the retrieval layer, not after the fact) to finer granularity. Filtering *after* retrieval means restricted documents were still processed by the similarity search and briefly existed in an intermediate result set — a weaker security posture that can leak information through side channels (timing, logs, ranking behavior) even if the final response is scrubbed. Applying the permission filter as part of the retrieval query itself — so restricted documents are structurally excluded as candidates — is the more robust design, mirroring how you'd want row-level security enforced in a database rather than filtered client-side.

**Why not A:** Post-retrieval filtering is a real anti-pattern precisely because "never candidates" is a stronger guarantee than "candidates that get removed later" — the removal step can be buggy, forgotten in some code path, or leak information before it runs.

**Why not C:** Giving up on document-level permissions because it's "too complex" isn't a real answer to a design question — it dodges the actual requirement rather than solving it, which is itself a red flag in an interview setting.

**Why not D:** Encryption addresses data-at-rest confidentiality, not query-time access control — a user could still have documents retrieved and decrypted for a query they shouldn't have access to if permission filtering isn't applied at the retrieval layer; encryption and access control solve different problems.

**Interviewer's likely follow-up:** *"How would you model complex permissions — like hierarchical folders or group-based ACLs — as a metadata filter efficiently?"* (Answer: typically pre-compute a flattened, queryable representation of "which document IDs / ACL groups can this user see" at query time — e.g., resolving group membership to a set of allowed tags or IDs — so the retrieval-time filter stays a simple, fast metadata lookup rather than doing complex permission resolution inline during vector search.)
</details>

### Q4.32 · Evaluating chunking strategy changes · [Design]

A team wants to A/B test a new chunking strategy against their current one before rolling it out. What should the evaluation actually measure?

- **A.** Only whether the new strategy produces fewer total chunks (efficiency)
- **B.** End-to-end retrieval and answer-quality metrics on a labeled eval set — recall@k, precision@k/NDCG for retrieval, plus downstream answer correctness/groundedness — comparing both strategies on the *same* set of representative queries, not just intermediate proxies like chunk count or average chunk size
- **C.** Whether the new strategy runs faster at ingestion time
- **D.** Whether the new chunks are more readable to a human reviewer

<details>
<summary>Answer</summary>

**B**

**Why B:** Chunking strategy is a means to an end — better retrieval and better final answers — so the evaluation needs to measure that end directly, on a labeled eval set with real representative queries, comparing both strategies' effect on retrieval metrics and downstream answer quality. Intermediate properties like chunk count or ingestion speed are not what you actually care about; they can look fine while retrieval quality gets worse, or vice versa.

**Why not A:** Chunk count is a cheap-to-compute proxy that has no guaranteed relationship to retrieval or answer quality — fewer chunks could mean better semantic boundaries, or it could just mean everything got mashed into fewer, worse chunks. It's the wrong thing to optimize for directly.

**Why not C:** Ingestion speed is an operational cost consideration, not a quality signal — a faster chunker that produces worse chunks is a bad trade, and this metric alone can't distinguish that.

**Why not D:** Human readability of chunks is a nice sanity check but not the actual target — a chunk can look readable to a human and still perform poorly in retrieval (e.g., missing the specific term a query would use), or look awkward but retrieve and answer perfectly well.

**Interviewer's likely follow-up:** *"How would you build the labeled eval set in the first place if you don't have one yet?"* (Answer: source real historical queries where you know or can determine the correct answer/source document — from support tickets, logged user questions, or SME-curated question-document pairs — covered in more depth in file 06's golden dataset material.)
</details>

### Q4.33 · RAG failure attribution with a reranker in the pipeline · [Applied]

A pipeline is: retrieve top 30 → rerank → pass top 5 to LLM. An answer is wrong. Where do you look first, and in what order?

- **A.** Assume it's always a generation failure since reranking should have fixed retrieval issues
- **B.** Check in pipeline order: was the correct chunk even in the initial top 30 (retrieval failure if not)? If yes, did the reranker correctly rank it into the final top 5 (reranking failure if it was pushed out)? If yes, did generation actually use it correctly (generation/groundedness failure, per Q4.11, if the chunk made it all the way to the model but was misused or ignored)?
- **C.** Assume it's always a retrieval failure since that's the earliest stage
- **D.** Re-run the same query multiple times until it produces a correct answer, then move on

<details>
<summary>Answer</summary>

**B**

**Why B:** With three stages in the pipeline, systematic debugging means checking them in order, because each stage's output is the next stage's input — there's no point debugging generation if the correct chunk never survived retrieval or reranking. First confirm the gold chunk was in the initial candidate set (retrieval); if so, confirm the reranker kept it in the final top 5 rather than incorrectly demoting it (reranking); only if it definitely reached the model do you investigate the generation stage as in Q4.11. This ordered attribution is what makes debugging tractable instead of guessing.

**Why not A:** Assuming it's always a generation failure skips checking whether the information ever actually reached the model — you could spend time "fixing" a prompt when the real bug is that the reranker demoted the right chunk.

**Why not C:** Assuming it's always retrieval ignores that a well-functioning retrieval stage can still be undone by a buggy reranker or a generation step that ignores good context — jumping straight to "improve recall@k" would misdiagnose a reranking or generation bug.

**Why not D:** Re-running the same query hoping for a different (correct) answer doesn't diagnose anything — it treats a systematic pipeline bug as random noise, and even if a retry happens to work, the underlying bug remains for the next similar query.

**Interviewer's likely follow-up:** *"How would you build tooling to make this three-stage attribution fast, rather than manually checking each time?"* (Answer: log the full pipeline trace per request — initial retrieved set with scores, post-rerank set with scores, and final generated answer with used context — so any failing example can be attributed to a stage in seconds by inspecting the trace, rather than re-running debug queries manually. This is the kind of observability covered further in file 06.)
</details>

### Q4.34 · Cross-lingual RAG · [Design]

A Singapore-based company's knowledge base has documents in English and Mandarin, and users query in either language interchangeably. What's a key design consideration for the embedding/retrieval layer?

- **A.** Maintain two completely separate pipelines with no cross-language retrieval, and route each query only to same-language documents
- **B.** Use a multilingual embedding model that places semantically equivalent content from different languages close together in vector space, so an English query can retrieve relevant Mandarin documents (and vice versa) rather than being artificially limited to same-language matches
- **C.** Translate all documents to English at query time using the LLM before every retrieval
- **D.** This isn't solvable with current retrieval technology — require users to specify their query language and only search that language's documents

<details>
<summary>Answer</summary>

**B**

**Why B:** Multilingual embedding models are specifically trained so that semantically equivalent text in different languages maps to nearby points in vector space — this lets a query in one language retrieve genuinely relevant content authored in another, which is exactly what's needed when a knowledge base itself is mixed-language and users don't reliably match their query language to the document's language. This is a real, solved capability, not a hypothetical.

**Why not A:** Restricting to same-language retrieval would miss relevant content whenever a user's query language doesn't match the document's language — a common, realistic case in a bilingual Singapore context, and exactly the gap a multilingual embedding model is meant to close.

**Why not C:** Translating every document at query time is expensive (extra LLM calls on every request) and unnecessary — multilingual embedding models solve the matching problem without needing to translate content at all; translation, if needed, belongs at ingestion time (once) rather than at query time (repeatedly).

**Why not D:** This defeats the purpose of the requirement (users query interchangeably) and is also simply not true — multilingual retrieval is a well-established, practically solvable capability with existing embedding models.

**Interviewer's likely follow-up:** *"What would you still need to validate before trusting a multilingual embedding model in production?"* (Answer: evaluate retrieval quality separately per language pair on your own labeled queries — multilingual models vary significantly in how well they handle any given language pair, and general benchmark performance doesn't guarantee good performance specifically on, say, English-query-to-Mandarin-document retrieval for your domain.)
</details>

### Q4.35 · RAG system-design synthesis · [Design]

You're asked to design RAG for a Singapore fintech's internal compliance-document assistant: documents update frequently as regulations change, access must be scoped by department, some documents are highly sensitive, and wrong answers carry real regulatory risk. What should shape the design most?

- **A.** Maximizing retrieval speed above all other considerations, since compliance staff want fast answers
- **B.** A design centered on correctness and auditability over raw speed: robust re-indexing for frequently-updated documents (Q4.14), department-level access control enforced at the retrieval layer (Q4.15/Q4.31), strong groundedness/citation validation given the regulatory cost of wrong answers (Q4.20), and full request/response/retrieval logging for audit trails — accepting some added latency as the right tradeoff given the stakes
- **C.** Skipping metadata filtering since all compliance staff are equally trusted internal employees
- **D.** Using the largest possible k value on every query to maximize the chance of not missing anything

<details>
<summary>Answer</summary>

**B**

**Why B:** This question is really asking you to synthesize the file's threads under real constraints. Frequent regulatory updates demand the re-indexing rigor from Q4.14. Department-scoped access control is a hard security requirement, not optional, per Q4.15/Q4.31. Given that wrong answers carry regulatory risk, groundedness and citation validation (Q4.20) stop being a nice-to-have and become central to the design — an ungrounded, confidently-wrong compliance answer is a genuinely dangerous failure mode. And full logging is needed for audit trails, which compliance and regulatory contexts specifically require. Speed matters, but correctness and auditability should dominate the design given the stated stakes.

**Why not A:** Optimizing purely for speed while the question explicitly flags regulatory risk from wrong answers gets the priority ordering backwards — a fast wrong answer in this domain is worse than a slightly slower correct, well-cited one.

**Why not C:** "Internal employees are equally trusted" is exactly the kind of assumption that gets flagged in a security review — departmental access boundaries typically exist for real compliance/need-to-know reasons even among internal staff, and dropping metadata filtering here ignores an explicitly stated requirement.

**Why not D:** Maximizing k blindly reintroduces the noise/precision/latency costs from Q4.24 without addressing any of the actual stated requirements (freshness, access control, correctness, auditability) — it's a brute-force non-answer to a design question that has real, specific constraints to engineer around.

**Interviewer's likely follow-up:** *"How would you communicate the confidence level of an answer to a compliance officer, given they can't just trust it blindly?"* (Answer: surface retrieved source citations directly alongside the answer so a human can verify against the original text, potentially with a groundedness/confidence score, and design the UX to make "I'm not certain, here's what I found" a normal, expected output rather than something the system tries to avoid — matching the sponsorship-adjacent principle that shipping honestly-uncertain answers beats shipping confidently wrong ones.)
</details>

---

## Explain prompts

### E4.1 · Explain: RAG vs fine-tuning vs long-context

**Prompt:** *"Someone on your team wants to fine-tune a model to 'teach it' your company's product documentation, since that seems like the natural way to make it know your product. Walk me through how you'd respond."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies that fine-tuning changes model *behavior/style*, not reliably *facts* — it doesn't guarantee the model will accurately recall specific documentation content
- [ ] Points out that documentation changes over time, and fine-tuning doesn't handle that gracefully (needs retraining every update)
- [ ] Proposes RAG as the better fit for "know facts from a document set that changes"
- [ ] Notes that fine-tuning has legitimate uses (tone, format, task-specific behavior) — doesn't dismiss it entirely, just for this use case
- [ ] Mentions long-context as a third option worth considering for small, static doc sets

**Bonus — signals strength:**
- [ ] Mentions that RAG gives natural citation/traceability (you can point to the source chunk), which fine-tuned "knowledge" can't
- [ ] Raises hallucination risk — a fine-tuned model can still confidently state wrong facts, same as a base model, with no retrieval to ground it
- [ ] Suggests a hybrid: fine-tune for tone/format, RAG for facts

**Red flags — deduct:**
- [ ] Says "fine-tuning teaches the model facts" without qualification
- [ ] Dismisses fine-tuning as always wrong/useless
- [ ] Can't articulate why document *updates* specifically favor RAG

**Score: ___ / 5**

**Model answer:**
So the first thing I'd push back on gently is the assumption baked into "teach it the docs" — fine-tuning doesn't really work like teaching a person facts. It nudges the model's behavior and style more reliably than it reliably injects specific factual recall, and honestly even when it does pick up some facts, you've got no guarantee it's not hallucinating something plausible-sounding instead. And our docs change — if we fine-tune today, we're stale the moment someone edits a page, and re-tuning every time isn't realistic. RAG just fits this so much better: we keep the docs as a live, searchable index, and the model looks stuff up per question, so updates show up immediately, and we get citations back to the actual source, which matters a lot for trust. I wouldn't throw fine-tuning out completely — if we wanted the assistant to always answer in our brand voice or a specific format, that's a legit fine-tuning use case. But "know current product facts" is RAG's job, not fine-tuning's.
</details>

### E4.2 · Explain: chunking tradeoffs

**Prompt:** *"Walk me through how you'd decide on a chunking strategy for a new RAG pipeline — not just 'what chunk size,' but your actual decision process."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Starts from the content type/structure, not a default number (docs vs code vs tables need different handling)
- [ ] Names the core tradeoff: smaller chunks = more precise but risk severing context; larger chunks = more complete but noisier
- [ ] Mentions overlap as a lever, with its own cost (redundancy)
- [ ] States this should be validated empirically against an eval set, not picked once and assumed correct
- [ ] Mentions considering downstream k and total context budget together with chunk size, not chunk size in isolation

**Bonus:**
- [ ] Mentions semantic/structure-aware chunking as an option, with its cost (more complex, more compute)
- [ ] Raises the "small-to-big" hierarchical retrieval pattern
- [ ] Notes chunking should probably be revisited if the corpus's content type changes significantly

**Red flags:**
- [ ] States a specific chunk size as a universal rule with no reasoning
- [ ] Treats chunking as a one-time decision that never needs revisiting
- [ ] Ignores content-type differences entirely

**Score: ___ / 5**

**Model answer:**
Honestly I wouldn't start with a number, I'd start by actually looking at the content. Is this dense prose, is it code, tables, mixed? Because that changes everything — a 300-token chunk works fine for FAQ-style prose but'll mangle a function or a table row. Once I know the content shape, the real tradeoff I'm balancing is: smaller chunks give you precision, less noise per match, but you risk cutting a thought in half; bigger chunks keep more context together but each match drags in more irrelevant filler, and it costs more per retrieval. Overlap helps with the cutting-in-half problem but it's not free either — you end up storing and sometimes retrieving near-duplicate chunks. So what I'd actually do is pick a reasonable starting point, build a small labeled eval set of real queries with known right answers, and then tune chunk size and overlap against retrieval metrics instead of guessing — and I'd expect to revisit it if the corpus composition changes later.
</details>

### E4.3 · Explain: hybrid search value

**Prompt:** *"A stakeholder asks why you need 'two search systems' — dense and BM25 — when you already have a good embedding model. Explain hybrid search to them."*

**Target:** 45–75 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Explains dense embeddings capture meaning/semantics, not exact terms
- [ ] Explains BM25/sparse captures exact lexical matches, which embeddings can miss (rare terms, codes, names)
- [ ] Gives a concrete example of each method's blind spot
- [ ] States that combining them (hybrid) covers both weaknesses, generally outperforming either alone
- [ ] Avoids jargon overload — explains it in plain terms since the audience is a stakeholder, not an engineer

**Bonus:**
- [ ] Mentions how scores get combined (RRF) at a high level without diving into math
- [ ] Frames it as "belt and suspenders," not "one is broken"

**Red flags:**
- [ ] Implies BM25 is outdated/obsolete
- [ ] Can't give a concrete example of a case where one method fails
- [ ] Answer is too technical for a non-technical stakeholder audience, ignoring who's actually asking

**Score: ___ / 5**

**Model answer:**
Good question — think of it this way: our embedding search is great at understanding *meaning*. If someone searches "how do I get my money back," it'll find our refund policy even if the policy doesn't use those exact words. But it's actually kind of bad at exact stuff — like if someone searches an exact order number or a product code, the embedding model doesn't really "understand" that string, it might match on things that just look similar. That's where the second system, BM25, comes in — it's old-school keyword matching, and it's really good at exactly that case: exact terms, codes, names. So it's not that our embedding search is broken, it's that the two systems are good at different things, and running both and combining the results covers each other's blind spots. It's belt and suspenders, not redundancy.
</details>

### E4.4 · Explain: diagnosing a RAG failure live

**Prompt:** *"A user reports a wrong answer from our RAG chatbot. You have access to the full pipeline logs. Walk me through your diagnostic process, step by step."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Starts by checking whether the correct information was in the retrieved chunks at all
- [ ] Explicitly separates "retrieval failure" from "generation failure" as the first branch point
- [ ] If reranking is present, checks it as a separate potential failure stage
- [ ] If retrieval failed, considers chunking/embedding/query-formulation as possible root causes
- [ ] If retrieval succeeded but generation failed, considers prompt template bugs, conflicting context, or hallucination despite grounding

**Bonus:**
- [ ] Mentions this process should ideally be systematized (logging/tracing) rather than manual each time
- [ ] Notes checking whether this is a one-off or a pattern before over-fixing on a single example

**Red flags:**
- [ ] Jumps straight to "retrain the embedding model" without checking what actually happened
- [ ] Doesn't separate retrieval from generation as distinct failure stages
- [ ] Treats every bad answer the same way regardless of where it broke

**Score: ___ / 5**

**Model answer:**
First thing I'd do, before touching anything, is pull up the trace for that exact request — what got retrieved, what got reranked if we have that stage, and what the model actually generated. And the very first branch point is: was the right information even in the retrieved chunks? If it wasn't, that's a retrieval-stage bug — could be chunking cut the info awkwardly, could be the embedding didn't match the query well, could be the query itself was too vague and needed rewriting. If the right info *was* retrieved but the answer's still wrong, that's a completely different investigation — now I'm looking at the prompt template, is the context actually getting inserted where I think it is, is there conflicting info in the other retrieved chunks confusing things, or is the model just hallucinating despite having the right grounding. I wouldn't jump to "the embedding model's bad" without first confirming retrieval actually failed — that's a common overreaction that fixes the wrong thing. And before I go rearchitect anything based on one report, I'd check if this is a one-off or part of a pattern.
</details>

### E4.5 · Explain: multi-tenant RAG security

**Prompt:** *"Explain how you'd design access control for a RAG system serving multiple customers, and why you'd reject the 'just tell the model not to' approach."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States that access control must be enforced at the retrieval layer, not the prompt/instruction layer
- [ ] Explains why prompt instructions aren't a security boundary (can be manipulated, model can err)
- [ ] Mentions metadata filtering or separate indexes as concrete mechanisms
- [ ] Notes this should apply before similarity ranking, not as a post-filter
- [ ] Connects this to real consequences — cross-tenant data leakage is a serious incident, not a minor bug

**Bonus:**
- [ ] Mentions testing this specifically (automated tests for tenant isolation)
- [ ] Extends to document-level, not just tenant-level, permissions

**Red flags:**
- [ ] Suggests a system prompt instruction as sufficient on its own
- [ ] Doesn't distinguish pre-retrieval filtering from post-retrieval filtering
- [ ] Treats this as a low-stakes concern

**Score: ___ / 5**

**Model answer:**
So the wrong way to do this — and I've seen it suggested — is just telling the model in the system prompt, "only talk about this customer's documents." That's not a security boundary, that's a request, and the model can get that wrong, or get manipulated into getting it wrong, and now you've leaked another customer's data. That's a genuinely bad incident, not a minor bug. The way I'd actually do it is enforce it at the retrieval layer — either separate indexes per tenant, which is the strongest isolation, or a mandatory metadata filter on tenant ID that gets applied before the similarity search even runs, so another tenant's documents are never candidates in the first place, not something we're trusting the model to ignore after the fact. And I'd want automated tests that seed fake data for two tenants and assert cross-tenant queries never return the wrong tenant's stuff, running in CI, not just something I manually check once and hope holds.
</details>

### E4.6 · Explain: groundedness and hallucination in RAG

**Prompt:** *"Even with retrieval working perfectly, RAG systems can still hallucinate. Explain how, and what you'd do about it."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Clarifies that "retrieval worked" and "answer is grounded" are different things
- [ ] Gives a concrete mechanism: model ignores/misuses provided context, or fabricates beyond what's given
- [ ] Mentions a mitigation: faithfulness/groundedness checking
- [ ] Mentions a mitigation: prompting the model to explicitly say when it can't answer from context
- [ ] Notes this can't be reduced to zero, only mitigated

**Bonus:**
- [ ] Mentions citation validation as a concrete technique
- [ ] Distinguishes this from a retrieval-stage bug (ties back to E4.4's diagnostic split)

**Red flags:**
- [ ] Claims RAG eliminates hallucination
- [ ] Can't explain the mechanism, just states the fact that it happens
- [ ] Offers no concrete mitigation

**Score: ___ / 5**

**Model answer:**
Yeah, this trips people up — a lot of people assume RAG just solves hallucination because "the model has the real info now." But having the right info in context doesn't guarantee the model uses it correctly. It can still ignore what's there and go with something more "typical-sounding," or blend the real context with something it's making up, or just misread which part of the context actually answers the question. So what I'd do is add a groundedness check — basically a pass, could be another LLM call, that checks whether each claim in the answer is actually backed by what was retrieved. I'd also prompt it explicitly to say "I don't have enough information" instead of always trying to produce a confident answer, because a lot of hallucination comes from the model feeling like it has to answer something. None of this gets you to zero — you're reducing the rate, not eliminating it — but it's a real, measurable improvement over just hoping retrieval alone fixes it.
</details>

### E4.7 · Explain: retrieval metrics to a non-ML stakeholder

**Prompt:** *"Your PM asks 'how do we know if the search part of our RAG system is actually good?' without any ML background. Explain recall@k and precision@k in plain terms, and why you'd track both."*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Explains recall@k in plain terms (are we finding the right stuff, somewhere in what we return)
- [ ] Explains precision@k in plain terms (of what we return, how much is actually useful)
- [ ] Gives a concrete, relatable example distinguishing the two
- [ ] Explains why tracking only one is misleading
- [ ] Avoids unexplained jargon given the non-technical audience

**Bonus:**
- [ ] Ties the metrics back to a concrete product outcome ("if recall's low, users get 'I don't know' answers; if precision's low, answers get confused by junk")
- [ ] Mentions this requires a labeled test set to actually measure

**Red flags:**
- [ ] Uses unexplained ML jargon with a stated non-technical audience
- [ ] Conflates the two metrics
- [ ] Can't give a concrete example

**Score: ___ / 5**

**Model answer:**
Sure — so think of it like this. Recall is basically "did we find the needle in the haystack at all." If the right document exists somewhere and our system found it in the results it gave back, that's a recall win. Precision is different — it's "of the stuff we handed back, how much of it was actually useful," versus just noise mixed in. And you actually need both, because you can have great recall and terrible precision — like, we found the right thing, but buried it under 20 irrelevant results — and that still gives the user a bad experience because the model's now sifting through a mess. Or the opposite: everything we return is relevant, but we're missing stuff entirely, so users get "I don't know" way more than they should. Tracking just one would hide the other kind of problem, so we look at both, measured against a set of real questions where we already know the right answer.
</details>

### E4.8 · Explain: production-scale RAG gap

**Prompt:** *"You've built a RAG demo over a static PDF. An interviewer asks: 'what changes when this needs to run in production against a live, constantly-updating, multi-tenant document set?' Walk through it."*

**Target:** 90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Names document freshness/re-indexing as a new problem that didn't exist with a static PDF
- [ ] Names access control/multi-tenancy as a new problem
- [ ] Mentions monitoring/eval at scale — you can't manually check every answer anymore
- [ ] Mentions cost/latency now actually matters at real query volume (ties to file 02)
- [ ] Frames this as the real gap between "demo" and "production" rather than treating them as the same problem at different scale

**Bonus:**
- [ ] Mentions the need for observability/tracing per request to debug at scale
- [ ] Mentions handling ingestion failures/partial updates gracefully

**Red flags:**
- [ ] Says "nothing really changes, just point it at more documents"
- [ ] Doesn't mention access control at all
- [ ] Treats this as purely a scale/performance problem, ignoring correctness and security dimensions

**Score: ___ / 5**

**Model answer:**
Honestly, almost everything changes — a static-PDF demo kind of hides all the real problems. First, the document set isn't static anymore, so I need a real re-indexing pipeline — something that detects changes and updates the index without leaving stale and fresh versions both floating around at once. Second, multi-tenant means I now have a hard security requirement — customer A's documents can never leak into customer B's results, and that has to be enforced at the retrieval layer, not just hoped for. Third, at real scale I can't manually eyeball whether answers are good anymore — I need actual evals, metrics, some kind of ongoing monitoring, because I won't see most of the traffic myself. And cost and latency, which barely mattered on a demo running a few queries, now genuinely matter — every extra chunk or rerank step is real money and real time multiplied across actual query volume. So it's not really "the same system but bigger" — it's a different set of engineering problems layered on top of the same core retrieval idea.
</details>

### E4.9 · Explain: when NOT to use RAG

**Prompt:** *"Give me a case where a team reached for RAG and it was the wrong architectural choice. What should they have done instead?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Gives a concrete example (e.g., computing an exact value like a balance, or answering from a small fixed doc set)
- [ ] Explains *why* RAG is a poor fit for that case (semantic retrieval isn't exact/complete/computational)
- [ ] Names the better alternative (tool/API call, long-context, or plain lookup)
- [ ] Shows understanding that RAG is one tool among several, not a default

**Bonus:**
- [ ] Connects to the broader principle of matching architecture to the actual nature of the task (echoes "when not to use an agent" reasoning)
- [ ] Mentions cost as a reason to avoid RAG when a simpler approach works

**Red flags:**
- [ ] Can't come up with a concrete example
- [ ] Treats RAG as always the right choice for "knowledge" questions
- [ ] Doesn't propose a concrete alternative

**Score: ___ / 5**

**Model answer:**
Yeah — a good one is when someone tries to use RAG to answer something that needs an exact, current, computed value, like "what's my account balance." I've seen the instinct to just embed transaction records and retrieve against that, and it's the wrong tool, because similarity search isn't guaranteed to find *every* relevant transaction, and it definitely can't add them up. What they actually needed was a straightforward database query or API call — a tool call, not a retrieval step — to fetch and compute the real number deterministically. RAG's great for "find relevant unstructured content I don't know the exact wording of," but it's the wrong instinct the moment the task is really "look up and compute an exact fact," and I think that mistake comes from treating RAG as the default answer to anything involving "the model needs data" instead of actually asking what kind of data operation is needed.
</details>

### E4.10 · Explain: reranking cost/benefit

**Prompt:** *"A team is deciding whether to add a reranking stage. Talk through the tradeoff you'd walk them through."*

**Target:** 60 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] States the benefit: more precise ranking of already-retrieved candidates, cutting noise
- [ ] States the cost: added latency and compute from running a heavier model
- [ ] Notes reranking can't fix a retrieval miss — the chunk has to be in the candidate set already
- [ ] Frames it as most worth it when the initial candidate set (top-k) is large/noisy enough that precision is a real problem
- [ ] Suggests measuring the actual impact rather than adding it by default

**Bonus:**
- [ ] Mentions typical implementation (cross-encoder on top of initial retrieval)
- [ ] Mentions this is one of several lower-cost alternatives to consider (e.g., better initial k tuning) before assuming reranking is needed

**Red flags:**
- [ ] Presents reranking as free or as a strict improvement with no cost
- [ ] Implies reranking can recover chunks retrieval missed
- [ ] No mention of measuring before deciding

**Score: ___ / 5**

**Model answer:**
So the honest tradeoff is: reranking makes your final result set more precise — it's a heavier, more accurate model looking at the query and each candidate together, so it's much better at ranking than the cheap initial retrieval step. But it's not free, it's an extra model call in the critical path, so you're adding latency and cost on every single query. And it's important to be clear on what it can't do — if the right chunk never made it into your initial top-k at all, reranking can't magically produce it, it can only re-order what's already there. So I'd want to know first: is our initial retrieval finding the right stuff but ranking it poorly? If yes, reranking is probably worth the cost. If we're actually missing relevant chunks entirely, reranking won't fix that, and we should look at chunking or the retrieval method itself first. I wouldn't add it by default — I'd measure where the actual problem is first.
</details>

### E4.11 · Explain: handling frequently-updating documents

**Prompt:** *"Explain the engineering challenge of keeping a RAG index in sync with a knowledge base that's edited constantly, and what could go wrong if you get it wrong."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Explains the core challenge: detecting changes reliably and re-indexing promptly
- [ ] Mentions the risk of stale/old chunks coexisting with new ones for the same document
- [ ] Names a concrete mechanism for change detection (webhook, polling with hashes, CDC)
- [ ] Emphasizes atomic replacement, not partial update, of a document's chunks
- [ ] Gives a concrete consequence of getting it wrong (contradictory or outdated answers)

**Bonus:**
- [ ] Discusses acceptable lag/eventual consistency as a reasonable tradeoff if bounded and known
- [ ] Mentions monitoring/alerting on ingestion pipeline failures

**Red flags:**
- [ ] Treats "just re-index everything periodically" as sufficient without addressing the coexistence/staleness risk
- [ ] Doesn't mention atomicity of the update
- [ ] No concrete failure consequence given

**Score: ___ / 5**

**Model answer:**
The tricky part isn't really "how do we re-embed a document" — that's the easy bit. The actual challenge is making sure the update is reliable and atomic. If someone edits an article and our re-indexing lags, or worse, half-updates — old chunks from the previous version and new chunks from the edited version both sitting in the index at once — now a query can retrieve the outdated fact right next to its correction, and the model has no way to know which one's current. That's worse than just being slightly stale, that's actively contradictory. So I'd want a reliable change-detection mechanism, could be a webhook from the CMS, could be polling with content hashes to catch anything the webhook missed, and then when a document changes, fully retire its old chunks and swap in the new ones as close to atomically as I can manage, so there's never a window where both versions are live. Some lag is fine — a few minutes of eventual consistency is a reasonable tradeoff — but silent inconsistency, where old and new both exist indefinitely, is the failure mode that actually damages trust in the system.
</details>

### E4.12 · Explain: designing RAG under real constraints (synthesis)

**Prompt:** *"Design, out loud, a RAG system for a Singapore healthcare provider's internal clinical-guidelines assistant — guidelines update periodically, doctors across different departments need different guideline subsets, and a wrong or outdated answer has real patient-safety implications. Walk through your top priorities."*

**Target:** 90–120 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit:**
- [ ] Prioritizes correctness/groundedness given explicitly stated patient-safety stakes
- [ ] Addresses department-scoped access control at the retrieval layer
- [ ] Addresses guideline freshness/re-indexing given periodic updates, with attention to never surfacing outdated guidance silently
- [ ] Mentions citation/source visibility so a clinician can verify against the original guideline, not just trust the answer blindly
- [ ] Explicitly deprioritizes raw speed relative to correctness given the stated stakes

**Bonus:**
- [ ] Mentions logging/audit trail given the regulated, high-stakes domain
- [ ] Mentions an explicit "I don't have enough information, consult the full guideline / a specialist" fallback rather than forcing an answer
- [ ] Mentions versioning guidelines so a superseded guideline is clearly marked as such if still referenced anywhere

**Red flags:**
- [ ] Treats this like a generic RAG build with no acknowledgment of the elevated stakes
- [ ] Optimizes for speed/UX polish over correctness given explicit patient-safety framing
- [ ] No mention of access control despite it being explicitly stated as a requirement

**Score: ___ / 5**

**Model answer:**
Okay, given the stakes here — patient safety — I'd let correctness and verifiability drive almost every decision, even at the cost of some speed or convenience. First, access: doctors seeing only their department's relevant guidelines isn't optional, so that's a retrieval-layer filter, not a prompt instruction, same logic as any multi-tenant access problem. Second, freshness — guidelines update periodically, and the worst possible outcome here is a doctor getting outdated guidance and not knowing it's outdated, so I'd want re-indexing that's not just prompt but genuinely reliable, and I'd think hard about how to avoid outdated and current guidance ever coexisting silently in the index. Third, and this is the big one — I would not want this system just confidently answering. I'd want every answer to come with a visible citation back to the actual guideline text and version, so a clinician can verify it themselves rather than trusting the model's word for it, and I'd want the system comfortable saying "I don't have a clear answer, check the full guideline or escalate" rather than forcing a confident-sounding response. And I'd log everything — every query, every retrieved source, every answer — because in a regulated healthcare context you need that audit trail regardless of how well the system performs day to day.
</details>
