# Risk Attenuator API Reference

## Module: `corteX.security.attenuation`

Risk-based capability attenuation. As risk increases, agent capabilities automatically decrease. The agent becomes more careful with sensitive operations without any developer intervention. A novel corteX invention.

## Classes

### `RiskLevel`

**Type**: `str, Enum`

Risk classification with numeric ranges.

| Value | Range | Behavior |
|-------|-------|----------|
| `LOW` | 0.0 -- 0.3 | Full capabilities |
| `MEDIUM` | 0.3 -- 0.6 | Remove write, keep read + execute |
| `HIGH` | 0.6 -- 0.8 | Remove execute, keep read only |
| `CRITICAL` | 0.8 -- 1.0 | Read only + human approval required |

#### Class Methods

##### `from_score`

```python
@classmethod
def from_score(cls, score: float) -> RiskLevel
```

Map a numeric score (clamped to [0.0, 1.0]) to a `RiskLevel`.

---

### `AttenuationResult`

**Type**: `@dataclass`

Outcome of a risk-based attenuation operation.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `original_caps` | `CapabilitySet` | Capability set before attenuation |
| `attenuated_caps` | `CapabilitySet` | Capability set after attenuation |
| `risk_level` | `RiskLevel` | Computed risk level |
| `risk_score` | `float` | Raw numeric risk score (0.0-1.0) |
| `removed_actions` | `List[str]` | Human-readable list of stripped actions (e.g., `"tool:file_write:write"`) |
| `requires_approval` | `bool` | Whether human approval is needed to proceed |

---

### `RiskAttenuator`

Attenuate capabilities based on computed risk.

#### Methods

##### `attenuate_by_risk`

```python
def attenuate_by_risk(self, caps: CapabilitySet, risk: float) -> AttenuationResult
```

Attenuate capabilities according to a numeric risk score.

**Parameters**:

- `caps` (`CapabilitySet`): Current capability set.
- `risk` (`float`): Risk score in [0.0, 1.0].

**Returns**: `AttenuationResult` with the narrowed capability set.

**Attenuation rules by risk level**:

| Risk Level | Actions Removed | Actions Kept |
|------------|----------------|-------------|
| LOW | None | All |
| MEDIUM | `write`, `delete`, `admin` | `read`, `execute` |
| HIGH | `write`, `delete`, `admin`, `execute` | `read` |
| CRITICAL | Everything except `read` | `read` only + approval required |

##### `compute_risk`

```python
def compute_risk(self, action: str, context: Dict[str, Any]) -> float
```

Compute a composite risk score from multiple factors.

**Parameters**:

- `action` (`str`): The action/tool being considered.
- `context` (`Dict[str, Any]`): Context dict with optional keys:
    - `data_level` (`str`): Data classification (`"public"`, `"internal"`, `"confidential"`, `"restricted"`).
    - `confidence` (`float`): Agent confidence (0.0-1.0). Lower = higher risk.
    - `drift_score` (`float`): Goal drift score (0.0-1.0). Higher = higher risk.

**Returns**: `float` -- Composite risk in [0.0, 1.0].

**Risk factor weights**:

| Factor | Weight | Description |
|--------|--------|-------------|
| Tool type risk | 0.35 | Inherent risk of the tool |
| Data level risk | 0.25 | Classification of data involved |
| Confidence risk | 0.20 | `1.0 - confidence` |
| Drift risk | 0.20 | Goal drift contribution |

##### `get_risk_thresholds`

```python
def get_risk_thresholds(self) -> Dict[str, float]
```

Return the risk-level threshold boundaries.

---

## Tool Risk Table

Built-in inherent risk scores for common tool types:

| Tool | Risk Score |
|------|-----------|
| `shell` | 0.9 |
| `file_write` | 0.8 |
| `code_interpreter` | 0.7 |
| `email` | 0.7 |
| `database` | 0.6 |
| `browser` | 0.5 |
| `api_call` | 0.5 |
| `memory_write` | 0.4 |
| `file_read` | 0.2 |
| `search` | 0.1 |
| `memory_read` | 0.1 |
| (default) | 0.3 |

---

## Data Level Risk Mapping

| Data Level | Risk Contribution |
|------------|------------------|
| `public` | 0.0 |
| `internal` | 0.3 |
| `confidential` | 0.7 |
| `restricted` | 1.0 |

---

## Example

```python
from corteX.security.attenuation import RiskAttenuator
from corteX.security.capabilities import Capability, CapabilitySet

attenuator = RiskAttenuator()

caps = CapabilitySet([
    Capability("tool:shell", frozenset({"read", "execute"})),
    Capability("tool:file_write", frozenset({"read", "write", "execute"})),
    Capability("tool:search", frozenset({"read", "execute"})),
])

# Compute risk for a shell command on confidential data
risk = attenuator.compute_risk("shell", {
    "data_level": "confidential",
    "confidence": 0.5,
    "drift_score": 0.3,
})
# risk ~= 0.35*0.9 + 0.25*0.7 + 0.20*0.5 + 0.20*0.15 = 0.62

# Attenuate capabilities
result = attenuator.attenuate_by_risk(caps, risk=0.62)
print(f"Risk level: {result.risk_level.value}")  # "high"
print(f"Removed: {result.removed_actions}")
# ["tool:shell:execute", "tool:file_write:execute", "tool:file_write:write"]
print(f"Approval needed: {result.requires_approval}")  # False

# Critical risk
result = attenuator.attenuate_by_risk(caps, risk=0.9)
print(f"Approval needed: {result.requires_approval}")  # True
```

---

## See Also

- [Capability Set](./capabilities.md) -- Core capability-based security
- [Data Classifier](./classification.md) -- Data level classification feeding risk
- [Compliance Engine](./compliance.md) -- Regulatory compliance checks
