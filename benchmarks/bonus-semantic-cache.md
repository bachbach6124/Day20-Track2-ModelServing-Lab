# Bonus Challenge C8 — Semantic Caching

## Setup
- **Hardware**: Apple M1 Pro, 16GB RAM, macOS 25.4.0
- **Configuration**: Offline sweep simulating hit rate versus threshold using `sentence-transformers` style embedding logic.

## Numbers
- **Threshold Sweep Results**:
  - Threshold `0.70` - `0.95`: Hit rate `38%` (3/8 hits)
  - Hits detected on paraphrases:
    - Query: *"Can you define goodput@SLO?"* (hit on *"What is goodput at SLO?"*)
    - Query: *"Tell me what goodput@SLO is."* (hit on *"What is goodput at SLO?"*)
    - Query: *"Describe how PagedAttention works."* (hit on *"How does PagedAttention work?"*)

- **Compute/Latency Savings**:
  - LLM calls saved: 3 (38% reduction in generation workload)
  - Estimated latency saved: ~750ms of decode time.

## Analysis & Observations
- **How Semantic Caching Helps**: A semantic cache operates at Layer 1 (above the prefix cache and KV cache). It intercepts incoming queries and performs a cosine-similarity comparison against historically answered prompts. On a HIT, the pre-saved answer is returned instantly (0ms latency, zero GPU compute).
- **Paraphrase Quality**: Simple embeddings catch syntactic changes easily (e.g. "Describe how X works" vs "How does X work"). However, conceptual paraphrases with different terminology (e.g. "TTFT" vs "time to first token") require a high-quality embedding model, or they will result in a cache MISS.
- **Security Implications**: In multi-tenant environments, semantic caches can leak prompts/information across users via timing attacks. To secure this, the cache must be partitioned or salted per tenant.
