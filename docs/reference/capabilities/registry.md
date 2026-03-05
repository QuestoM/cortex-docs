# Capability Registry

::: corteX.capabilities.registry.CapabilityRegistry

## Module: `corteX.capabilities.registry`

The `CapabilityRegistry` is the central store for all capability descriptors within a session. It provides registration with automatic duplicate detection, search, and manifest export/import.

## Constructor

```python
CapabilityRegistry(
    name: str = "default",
    duplicate_threshold: float = 0.7,
    strict: bool = False,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"default"` | Registry name (used in manifests and logging) |
| `duplicate_threshold` | `float` | `0.7` | Minimum overall similarity score to flag as duplicate |
| `strict` | `bool` | `False` | If True, raises `ValueError` on duplicate detection |

## Methods

### Registration

#### `register(descriptor) -> Optional[DuplicateReport]`

Register a `CapabilityDescriptor`. Returns a `DuplicateReport` if a duplicate was found above the threshold, `None` otherwise. In strict mode, raises `ValueError` instead of returning the report.

#### `register_function(func, *, name, category, tags, provider_id) -> Tuple[CapabilityDescriptor, Optional[DuplicateReport]]`

Convenience method: fingerprints a Python callable, builds a descriptor, and registers it. Tags are auto-extracted from the function's docstring if not provided.

### Lookup

#### `query(name) -> Optional[CapabilityDescriptor]`

Exact name lookup. Returns `None` if not found.

#### `search(query, top_k=5) -> List[Tuple[CapabilityDescriptor, float]]`

Free-text search using tag and description word overlap. Returns up to `top_k` results sorted by descending relevance score.

#### `find_similar(candidate) -> List[DuplicateReport]`

Compare a candidate descriptor against all registered capabilities. Returns `DuplicateReport` entries sorted by descending similarity.

### Collection

#### `get_all() -> List[CapabilityDescriptor]`

Return all registered descriptors.

#### `remove(capability_id) -> bool`

Remove a capability by ID. Returns `True` if found and removed.

#### `clear()`

Remove all capabilities from the registry.

### Serialization

#### `to_manifest() -> Dict[str, Any]`

Export the full registry as a JSON-serializable dict. Includes all descriptors with their fingerprints, tags, and metadata.

#### `from_manifest(data) -> CapabilityRegistry` (classmethod)

Reconstruct a registry from a previously exported manifest dict.

## Similarity Scoring

The registry uses a **dual-signal** approach:

```
overall = (0.6 x structural) + (0.4 x semantic)
```

- **Structural** (`CapabilityFingerprinter`): SHA-256 hash of sorted parameter names + types from `inspect.signature()`. Exact match = 1.0, same count + return type = 0.7, same count = 0.4.
- **Semantic** (`_helpers.semantic_similarity`): Jaccard similarity of tags + description words.

## Types

### `CapabilityDescriptor`

```python
@dataclass
class CapabilityDescriptor:
    name: str
    description: str
    category: CapabilityCategory
    capability_id: str = field(default_factory=...)  # Auto-generated UUID
    semantic_tags: List[str] = field(default_factory=list)
    fingerprint: Optional[CapabilityFingerprint] = None
    provider_id: str = ""
    registered_at: str = ""  # ISO timestamp
    metadata: Dict[str, Any] = field(default_factory=dict)
```

### `CapabilityFingerprint`

```python
@dataclass
class CapabilityFingerprint:
    param_hash: str       # SHA-256 of sorted param signatures
    return_type: str      # Return type string
    param_count: int      # Number of parameters
    is_async: bool        # Whether the function is async
    structural_signature: str  # Human-readable canonical form
```

### `SimilarityScore`

```python
@dataclass
class SimilarityScore:
    structural: float  # 0.0 - 1.0
    semantic: float    # 0.0 - 1.0
    overall: float     # Weighted combination
```

### `DuplicateReport`

```python
@dataclass
class DuplicateReport:
    existing: CapabilityDescriptor
    candidate: CapabilityDescriptor
    score: SimilarityScore
    verdict: DuplicateVerdict
    explanation: str
```

### `DuplicateVerdict` (Enum)

| Value | Score Range | Meaning |
|-------|-----------|---------|
| `DEFINITE_DUPLICATE` | >= 0.85 | Almost certainly the same capability |
| `LIKELY_DUPLICATE` | >= 0.7 | Very similar, review recommended |
| `POSSIBLE_OVERLAP` | >= 0.5 | Some overlap, may be intentional |
| `UNIQUE` | < 0.5 | Sufficiently different |

### `CapabilityCategory` (Enum)

Values: `TOOL`, `SKILL`, `WORKFLOW`, `INTEGRATION`, `CUSTOM`

## Related Modules

- [`CapabilityAnalyzer`](analyzer.md) - Redundancy clustering, health reports, registry diffing
- [`CapabilityFingerprinter`](fingerprinter.md) - Structural fingerprinting from callables
- [`SessionCapabilityMixin`](../session.md) - Session integration (auto-registration, API)

## See Also

- [Capability Registry Concept](../../concepts/capability-registry.md) - Architecture overview and usage guide
