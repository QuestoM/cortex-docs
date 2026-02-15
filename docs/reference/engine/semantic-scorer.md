# Semantic Importance Scorer API Reference

## Module: `corteX.engine.semantic_scorer`

Vector embedding-based importance scoring for context window management. Scores text by relevance (to goal + context), novelty (vs. seen texts), and length factor. On-prem capable with pure numpy TF-IDF backend. Supports pluggable embedding backends.

---

## Classes

### `EmbeddingBackend`

**Type**: `Protocol (runtime_checkable)`

Abstract protocol for embedding backends.

| Method | Description |
|--------|-------------|
| `embed(text: str) -> List[float]` | Compute embedding vector for text |
| `embed_batch(texts: List[str]) -> List[List[float]]` | Batch embedding |
| `similarity(vec_a, vec_b) -> float` | Cosine similarity between vectors |
| `dimension` (property) | Embedding dimensionality |

### `TFIDFBackend`

**Type**: Class implementing `EmbeddingBackend`

Incremental TF-IDF backend (pure numpy). Evicts lowest-df terms at vocabulary cap.

#### Constructor

```python
TFIDFBackend(max_vocabulary: int = 5000)
```

#### Methods

##### `fit_partial`

```python
def fit_partial(self, texts: List[str]) -> None
```

Incrementally update vocabulary from new documents. Auto-evicts lowest-df terms when vocabulary exceeds `max_vocabulary`.

##### `embed`

```python
def embed(self, text: str) -> List[float]
```

Compute TF-IDF vector for a single text. TF = term count / total tokens. IDF = log(1 + n_docs / (1 + doc_freq)).

##### `similarity`

```python
def similarity(self, vec_a: List[float], vec_b: List[float]) -> float
```

Cosine similarity between two vectors. Handles dimension mismatch by zero-padding the shorter vector (vocabulary may grow between embeddings).

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `vocabulary_size` | `int` | Current vocabulary size |
| `n_docs` | `int` | Total documents processed |
| `dimension` | `int` | Vector dimensionality |

### `ScorerConfig`

**Type**: `@dataclass`

Configuration for the semantic importance scorer.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `backend_type` | `str` | `"tfidf"` | Backend type (`"tfidf"` or `"sentence-transformers"`) |
| `max_vocabulary` | `int` | `5000` | Maximum TF-IDF vocabulary size |
| `relevance_weight` | `float` | `0.6` | Weight for relevance score |
| `novelty_weight` | `float` | `0.3` | Weight for novelty score |
| `length_weight` | `float` | `0.1` | Weight for length factor |
| `min_text_length` | `int` | `10` | Minimum text length to score |
| `cache_embeddings` | `bool` | `True` | Cache computed embeddings |
| `cache_max_size` | `int` | `2000` | Maximum cache entries (LRU) |

---

### `SemanticImportanceScorer`

Scores text importance via vector similarity (relevance + novelty + length).

#### Constructor

```python
SemanticImportanceScorer(
    backend: EmbeddingBackend,
    goal_text: str = "",
    config: Optional[ScorerConfig] = None,
)
```

#### Methods

##### `set_goal`

```python
def set_goal(self, goal_text: str) -> None
```

Update the goal embedding for relevance scoring. Fits the backend on the goal text.

##### `score_relevance`

```python
def score_relevance(
    self, text: str, context_texts: Optional[List[str]] = None,
) -> float
```

How relevant is text to the goal + recent context? Returns [0.0, 1.0]. Computes cosine similarity against goal embedding and context centroid.

##### `score_novelty`

```python
def score_novelty(
    self, text: str, seen_texts: Optional[List[str]] = None,
) -> float
```

How novel is text vs. previously seen texts? `1.0` = completely novel, `0.0` = duplicate. Computed as `1 - max_similarity(text, seen)`.

##### `score_importance`

```python
def score_importance(
    self, text: str, goal: str = "",
    context: Optional[List[str]] = None,
) -> float
```

Combined importance score: `relevance * 0.6 + novelty * 0.3 + length * 0.1`. Returns [0.0, 1.0]. Weights are configurable via `ScorerConfig`.

##### `find_most_relevant`

```python
def find_most_relevant(
    self, query: str, candidates: List[str], top_k: int = 5,
) -> List[Tuple[int, float]]
```

Return `(index, score)` of top-k most relevant candidates, sorted descending by score.

##### `invalidate_cache`

```python
def invalidate_cache(self) -> None
```

Clear the embedding cache.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `goal_text` | `str` | Current goal text |
| `cache_size` | `int` | Number of cached embeddings |
| `backend` | `EmbeddingBackend` | The underlying backend |

---

## Factory Function

```python
def create_scorer(
    config: Optional[ScorerConfig] = None,
    goal_text: str = "",
) -> SemanticImportanceScorer
```

Create scorer with configured backend. Currently supports `"tfidf"` (built-in) and `"sentence-transformers"` (requires pip install).

---

## Usage Example

```python
from corteX.engine.semantic_scorer import create_scorer

scorer = create_scorer(goal_text="Build a REST API for user management")

# Score relevance
relevance = scorer.score_relevance(
    "Creating database migration for users",
    context_texts=["Setting up Flask routes", "Defining user model"],
)
print(f"Relevance: {relevance:.2f}")

# Score novelty
novelty = scorer.score_novelty(
    "Adding JWT authentication",
    seen_texts=["Creating user model", "Setting up routes"],
)
print(f"Novelty: {novelty:.2f}")

# Combined importance
importance = scorer.score_importance(
    "Implementing rate limiting middleware",
    context=["User model", "Routes", "Auth"],
)
print(f"Importance: {importance:.2f}")

# Find most relevant context items for compaction
relevant = scorer.find_most_relevant(
    "authentication", candidates=context_items, top_k=3,
)
for idx, score in relevant:
    print(f"  [{idx}] score={score:.2f}: {context_items[idx][:60]}")
```

---

## See Also

- [Context Compiler API](./context-compiler.md)
- [Content Prediction API](./content-prediction.md)
