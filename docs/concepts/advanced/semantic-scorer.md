# Semantic Scoring

The Semantic Importance Scorer replaces keyword overlap with vector-based similarity for scoring context relevance. Built on pure numpy TF-IDF (no scikit-learn dependency), it scores text along three dimensions: relevance to the goal, novelty versus seen content, and length appropriateness.

## What It Does

The scorer answers three questions about any piece of text:

1. **Relevance**: How related is this text to the agent's current goal and context?
2. **Novelty**: How different is this text from content the agent has already seen?
3. **Importance**: Weighted combination of relevance (60%), novelty (30%), and length (10%)

## Why It Matters

Keyword overlap ("how many words match?") fails on paraphrases, synonyms, and semantic similarity. TF-IDF vectors capture term importance across documents, providing a lightweight semantic signal without requiring external embedding models or GPU inference.

The scorer is used by the Cortical Context Engine to decide what to keep in working memory and what to compress or discard.

## How It Works

### TF-IDF Backend

The `TFIDFBackend` builds an incremental vocabulary with IDF weighting:

```python
from corteX.engine.semantic_scorer import TFIDFBackend

backend = TFIDFBackend(max_vocabulary=5000)

# Incrementally learn vocabulary from documents
backend.fit_partial(["Analyze quarterly sales revenue", "Generate financial report"])

# Embed a text as a TF-IDF vector
vec = backend.embed("Sales analysis for Q4")

# Compute cosine similarity between two vectors
sim = backend.similarity(vec, backend.embed("Revenue report for Q4"))
# ~0.6 (semantically related)
```

The backend automatically evicts lowest document-frequency terms when vocabulary exceeds the cap.

### Importance Scoring

```python
from corteX.engine.semantic_scorer import create_scorer

scorer = create_scorer(goal_text="Analyze Q4 financial performance")

# Relevance: how related to the goal?
relevance = scorer.score_relevance(
    "Q4 revenue grew 15% year-over-year",
    context_texts=["Previous quarter showed 12% growth"],
)
# ~0.75 (highly relevant to financial analysis goal)

# Novelty: how different from seen content?
novelty = scorer.score_novelty(
    "Q4 revenue grew 15% year-over-year",
    seen_texts=["Q4 revenue increased by 15% compared to last year"],
)
# ~0.15 (near-duplicate, low novelty)

# Combined importance
importance = scorer.score_importance(
    "New customer acquisition cost dropped 20%",
    goal="Analyze Q4 financial performance",
    context=["Revenue grew 15%", "Margins improved"],
)
# ~0.72 (relevant + novel + good length)
```

### Semantic Search

Find the most relevant candidates for a query:

```python
results = scorer.find_most_relevant(
    query="What were the key financial metrics?",
    candidates=[
        "Revenue: $2.4M (+15%)",
        "Weather forecast: sunny",
        "CAC dropped 20%",
        "Team lunch at noon",
        "Gross margin: 68%",
    ],
    top_k=3,
)
# [(0, 0.82), (4, 0.78), (2, 0.71)]
# Revenue, gross margin, and CAC -- correctly filtered
```

### Pluggable Backend

The `EmbeddingBackend` protocol allows swapping TF-IDF for richer embeddings:

```python
from corteX.engine.semantic_scorer import ScorerConfig, create_scorer

# Default: TF-IDF (pure numpy, no extra dependencies)
scorer = create_scorer(config=ScorerConfig(backend_type="tfidf"))

# Optional: sentence-transformers (requires pip install)
# scorer = create_scorer(config=ScorerConfig(backend_type="sentence-transformers"))
```

Any backend implementing `embed()`, `embed_batch()`, `similarity()`, and `dimension` works.

## Configuration

```python
from corteX.engine.semantic_scorer import ScorerConfig

config = ScorerConfig(
    backend_type="tfidf",       # or "sentence-transformers"
    max_vocabulary=5000,        # TF-IDF vocabulary cap
    relevance_weight=0.6,       # Weight for goal relevance
    novelty_weight=0.3,         # Weight for content novelty
    length_weight=0.1,          # Weight for text length
    min_text_length=10,         # Skip very short texts
    cache_embeddings=True,      # LRU cache for embeddings
    cache_max_size=2000,        # Max cached embeddings
)
```

## When It Activates

- **Context management**: The CCE uses importance scores to decide compression priority
- **Memory retrieval**: Semantic search finds relevant memories for the current step
- **Novelty gating**: Duplicate or near-duplicate content gets lower priority
- **Continuously**: Vocabulary grows incrementally as the agent processes new content

## API Reference

```python
from corteX.engine.semantic_scorer import (
    SemanticImportanceScorer,
    TFIDFBackend,
    EmbeddingBackend,
    ScorerConfig,
    create_scorer,
    tokenize,
)
```
