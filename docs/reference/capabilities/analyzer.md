# Capability Analyzer

::: corteX.capabilities.analyzer.CapabilityAnalyzer

## Module: `corteX.capabilities.analyzer`

The `CapabilityAnalyzer` inspects a `CapabilityRegistry` for redundancy, overlaps, and overall health. It provides union-find cluster detection, health scoring, pre-registration "did you mean..." suggestions, and cross-registry diffing.

## Constructor

```python
CapabilityAnalyzer(registry: CapabilityRegistry)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `registry` | `CapabilityRegistry` | The registry instance to analyze |

## Methods

### Redundancy Detection

#### `find_redundancy_clusters(threshold=0.6) -> List[RedundancyCluster]`

Group capabilities whose pairwise similarity exceeds `threshold` using union-find clustering. Returns a list of `RedundancyCluster` objects, each containing the grouped capabilities and a consolidation suggestion.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `threshold` | `float` | `0.6` | Minimum similarity to merge two capabilities into the same cluster |

The algorithm:

1. Computes pairwise similarity for all capabilities in the registry
2. Merges pairs above `threshold` using union-find with path compression
3. Groups merged indices into clusters and computes average intra-cluster similarity
4. Generates a human-readable consolidation suggestion per cluster

#### `generate_health_report() -> CapabilityHealthReport`

Run full redundancy analysis and produce a health report. Calls `find_redundancy_clusters()` internally and computes a health score based on the ratio of redundant pairs to total possible pairs.

Health score formula:

```
health_score = max(0.0, 1.0 - (redundant_pairs / total_pairs))
```

Also generates actionable recommendations based on score thresholds:

- >= 0.95: "Registry is healthy - minimal redundancy detected."
- >= 0.80: "Minor redundancy detected - review suggested clusters."
- < 0.80: "Significant redundancy detected - consolidation recommended."

### Pre-Registration Suggestions

#### `suggest_existing(name, description, tags=None) -> List[Tuple[CapabilityDescriptor, float]]`

Search for similar capabilities before registration ("did you mean..."). Returns up to 10 matches with scores, sorted descending. Only includes results with similarity > 0.1.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | - | Name of the candidate capability |
| `description` | `str` | - | Description of the candidate |
| `tags` | `Optional[List[str]]` | `None` | Semantic tags for matching |

### Cross-Registry Diffing

#### `diff_registries(other) -> Dict[str, Any]`

Compare the analyzer's registry against another `CapabilityRegistry`. Returns a dict with:

| Key | Type | Description |
|-----|------|-------------|
| `added` | `List[CapabilityDescriptor]` | Capabilities in `other` but not in self |
| `removed` | `List[CapabilityDescriptor]` | Capabilities in self but not in `other` |
| `overlapping` | `List[Dict]` | Capabilities in both, with similarity score and descriptions |
| `summary` | `str` | Human-readable summary (e.g. "3 added, 1 removed, 5 overlapping") |

## Types

### `RedundancyCluster`

```python
@dataclass
class RedundancyCluster:
    cluster_id: str                         # e.g. "cluster-0"
    capabilities: List[CapabilityDescriptor] # Members of this cluster
    avg_similarity: float                    # Average intra-cluster similarity
    suggested_consolidation: str             # Human-readable suggestion
```

### `CapabilityHealthReport`

```python
@dataclass
class CapabilityHealthReport:
    total_capabilities: int               # Total registered capabilities
    unique_capabilities: int              # Those not in any cluster
    redundancy_clusters: List[RedundancyCluster]
    duplicate_pairs: List[DuplicateReport]
    health_score: float                   # 0.0 - 1.0 (higher = healthier)
    recommendations: List[str]            # Actionable suggestions
```

## Similarity Scoring

The analyzer uses the same **dual-signal** approach as the registry:

```
overall = (0.6 x structural) + (0.4 x semantic)
```

- **Structural**: Uses `CapabilityFingerprinter.structural_similarity()` to compare parameter hashes, counts, and return types.
- **Semantic**: Uses `_helpers.semantic_similarity()` - Jaccard similarity of tags (60%) + description word overlap (40%).

Duplicate detection threshold: overall >= 0.5 flags a pair as a duplicate in health reports.

## Related Modules

- [`CapabilityRegistry`](registry.md) - Central store for capability descriptors
- [`CapabilityFingerprinter`](fingerprinter.md) - Structural fingerprinting from callables
- [`SessionCapabilityMixin`](../session.md) - Session integration (auto-registration, API)

## See Also

- [Capability Registry Concept](../../concepts/capability-registry.md) - Architecture overview and usage guide
