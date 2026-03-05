# Capability Registry

The **Capability Registry** is corteX's built-in system for tracking, fingerprinting, and deduplicating all capabilities (tools, skills, functions) registered in an agent session. It prevents the common problem of duplicate capabilities accumulating over time as agents evolve.

## The Problem: Capability Sprawl

As AI agent systems grow, capabilities multiply. Different developers add tools with overlapping functionality. Automated systems register similar capabilities under different names. Over time, an agent might have `search_documents`, `find_documents`, and `query_documents` - all doing essentially the same thing.

This causes:

- **Wasted tokens** - the LLM sees redundant tool descriptions in its context
- **Decision confusion** - the model struggles to choose between near-identical tools
- **Maintenance burden** - fixing a bug requires updating multiple overlapping implementations

## How It Works

The Capability Registry uses a **dual-signal similarity engine** that combines structural and semantic analysis:

```
Overall Score = (0.6 x Structural) + (0.4 x Semantic)
```

### Structural Fingerprinting

The `CapabilityFingerprinter` inspects each callable's `inspect.signature()` to extract:

- Parameter names and type annotations
- Parameter count and return type
- Whether the function is async
- A SHA-256 hash of sorted `name:type` pairs

Two functions with identical parameter signatures get a structural score of **1.0**. Same param count and return type scores **0.7**. Same count only scores **0.4**.

### Semantic Similarity

Compares descriptions, tags, and names using word overlap (Jaccard similarity). Functions named `create_ticket` and `create_helpdesk_ticket` with similar docstrings will score high semantically even if their parameter signatures differ slightly.

### Duplicate Verdicts

Based on the combined score, the registry classifies each match:

| Score Range | Verdict | Meaning |
|------------|---------|---------|
| >= 0.85 | `DEFINITE_DUPLICATE` | Almost certainly the same capability |
| >= 0.7 | `LIKELY_DUPLICATE` | Very similar - review recommended |
| >= 0.5 | `POSSIBLE_OVERLAP` | Some overlap - may be intentional |
| < 0.5 | `UNIQUE` | Sufficiently different |

## Usage

### Basic Registration

```python
from cortex import Engine

engine = Engine()
agent = engine.create_agent(model="gemini-2.5-pro")
session = agent.create_session()

# Tools are auto-registered when the session starts
# Check for potential duplicates:
health = session.get_capability_health()
print(health)
# {'total_capabilities': 12, 'categories': {'tool': 12},
#  'potential_duplicates': 1, ...}
```

### Manual Registration with Duplicate Detection

```python
def search_knowledge_base(query: str, limit: int = 10) -> list:
    """Search the knowledge base for relevant articles."""
    ...

descriptor, report = session.register_capability(search_knowledge_base)

if report:
    print(f"Duplicate detected: {report.explanation}")
    # "search_knowledge_base overlaps with find_kb_articles (score=0.82)"
```

### Strict Mode

In strict mode, the registry raises `ValueError` on duplicate registration - useful in CI/CD pipelines:

```python
from corteX.capabilities.registry import CapabilityRegistry

registry = CapabilityRegistry(strict=True, duplicate_threshold=0.7)
# Raises ValueError if a duplicate is detected
registry.register(descriptor)
```

### Health Reports and Analysis

The `CapabilityAnalyzer` provides deeper analysis using union-find clustering:

```python
from corteX.capabilities.analyzer import CapabilityAnalyzer

analyzer = CapabilityAnalyzer(session.capability_registry)

# Find redundancy clusters
clusters = analyzer.find_redundancy_clusters(threshold=0.6)
for cluster in clusters:
    print(cluster.suggested_consolidation)

# Full health report
report = analyzer.generate_health_report()
print(f"Health score: {report.health_score}")
print(f"Recommendations: {report.recommendations}")
```

### Pre-Registration Suggestions

Before adding a new capability, check if something similar already exists:

```python
suggestions = analyzer.suggest_existing(
    name="search_docs",
    description="Search documents by keyword",
)
for existing, score in suggestions:
    print(f"  {existing.name}: {score:.2f}")
```

### Registry Diffing

Compare two registries to see what changed between versions:

```python
diff = analyzer.diff_registries(other_registry)
print(diff["summary"])
# "3 added, 1 removed, 8 overlapping"
```

### Manifest Export/Import

Export the full registry for persistence or CI validation:

```python
manifest = session.get_capability_manifest()
# Save to JSON, compare in CI, or import elsewhere

restored = CapabilityRegistry.from_manifest(manifest)
```

## Architecture

```
corteX/capabilities/
    types.py            - CapabilityDescriptor, CapabilityFingerprint, SimilarityScore, DuplicateReport
    fingerprint.py      - CapabilityFingerprinter (SHA-256 structural hashing)
    registry.py         - CapabilityRegistry (registration + duplicate detection)
    analyzer.py         - CapabilityAnalyzer (clustering, health, diffing)
    _helpers.py         - Shared utilities (Jaccard, semantic similarity, explanations)

corteX/session/
    capabilities.py     - SessionCapabilityMixin (auto-registers tools, exposes API)
```

## Integration with Session

The `SessionCapabilityMixin` integrates the registry into every corteX `Session`:

- **Lazy initialization** - the registry is created on first access, not at session start
- **Auto-registration** - all tools from the agent's `ToolExecutor` are automatically fingerprinted and registered
- **Transparent API** - `session.register_capability()`, `session.find_capability()`, `session.get_capability_health()`, and `session.get_capability_manifest()` are available on every session

## Best Practices

1. **Set thresholds appropriately** - 0.7 is a good default. Lower for stricter dedup, higher for more permissive.
2. **Use strict mode in CI** - catch duplicates before they reach production
3. **Review health reports** - run `generate_health_report()` periodically to track capability hygiene
4. **Use suggest_existing()** - before adding a new tool, check if a similar one already exists
5. **Export manifests** - save manifests to version control for change tracking

## See Also

- [Tool Reputation](tool-reputation.md) - track tool success rates and reliability
- [Security Framework](security.md) - capability-based access control (different concept, same word)
