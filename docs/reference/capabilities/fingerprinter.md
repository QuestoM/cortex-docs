# Capability Fingerprinter

::: corteX.capabilities.fingerprint.CapabilityFingerprinter

## Module: `corteX.capabilities.fingerprint`

The `CapabilityFingerprinter` extracts and compares structural fingerprints for capabilities. A fingerprint captures the shape of a callable - its parameter names, types, arity, async-ness, and return type - enabling fast structural comparison without invoking the function.

## Class: `CapabilityFingerprinter`

All methods are static - no constructor or instance state required.

## Methods

### Fingerprint Extraction

#### `from_function(func) -> CapabilityFingerprint` (staticmethod)

Build a fingerprint by inspecting a Python callable using `inspect.signature()` and `typing.get_type_hints()`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `func` | `Callable` | Any Python callable (function, method, lambda) |

The extraction process:

1. Collects parameter names and type annotations (skipping `self`/`cls`)
2. Sorts parameters alphabetically by name
3. Builds a canonical signature string (e.g. `(async count:int, name:str) -> bool`)
4. Computes a SHA-256 hash of sorted `name:type` pairs for fast equality checks

#### `from_tool_wrapper(wrapper) -> CapabilityFingerprint` (staticmethod)

Build a fingerprint from an existing `ToolWrapper` object. Prefers the underlying function when available via `wrapper.func`; falls back to the wrapper's JSON Schema `parameters` property.

| Parameter | Type | Description |
|-----------|------|-------------|
| `wrapper` | `ToolWrapper` | A corteX tool wrapper instance |

Fallback behavior when no inspectable function exists:

- Reads `properties` from the JSON Schema stored on the wrapper
- Sorts property names alphabetically
- Uses the schema `type` field as the type annotation
- Defaults `return_type` to `"str"` and `is_async` to `False`

### Structural Comparison

#### `structural_similarity(a, b) -> float` (staticmethod)

Compare two fingerprints and return a similarity score in [0, 1].

| Parameter | Type | Description |
|-----------|------|-------------|
| `a` | `CapabilityFingerprint` | First fingerprint |
| `b` | `CapabilityFingerprint` | Second fingerprint |

Scoring rules (first match wins):

| Condition | Score | Meaning |
|-----------|-------|---------|
| Exact `param_hash` match | 1.0 | Structurally identical parameters |
| Same `param_count` AND `return_type` | 0.7 | Very similar shape |
| Same `param_count` only | 0.4 | Same arity, different types |
| Same `is_async` only | 0.1 | Only async flag matches |
| Otherwise | 0.0 | No structural similarity |

## Types

### `CapabilityFingerprint`

Defined in `corteX.capabilities.types`. See [`CapabilityRegistry`](registry.md) for the full type definition.

```python
@dataclass(frozen=True)
class CapabilityFingerprint:
    param_hash: str            # SHA-256 of sorted param signatures
    return_type: str           # Return type string (e.g. "bool", "None")
    param_count: int           # Number of parameters (excluding self/cls)
    is_async: bool             # Whether the function is async
    structural_signature: str  # Human-readable canonical form
```

The `frozen=True` makes fingerprints hashable and immutable - they can be used as dict keys or set members.

## Module-Level Helpers

### `_type_name(tp) -> str`

Private helper that returns a stable string representation of a type annotation. Handles raw strings, types with `__name__`, and falls back to `str()`.

## Related Modules

- [`CapabilityRegistry`](registry.md) - Central store that uses fingerprints for duplicate detection
- [`CapabilityAnalyzer`](analyzer.md) - Redundancy clustering and health reports using fingerprint similarity
- [`SessionCapabilityMixin`](../session.md) - Session integration (auto-registration, API)

## See Also

- [Capability Registry Concept](../../concepts/capability-registry.md) - Architecture overview and usage guide
