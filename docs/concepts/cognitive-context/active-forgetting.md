# Active Forgetting

The `ActiveForgettingEngine` implements deliberate memory removal as a cognitive performance tool. Not all forgetting is loss - removing outdated, contradicted, or poisonous memories improves agent reliability and frees token budget for more relevant information.

## How It Works

The engine evaluates memory items against five forgetting triggers, each producing a `ForgettingEvent` with full provenance for potential reversal.

### The Five Triggers

#### 1. Contradiction

Detects when a new fact directly contradicts an existing memory. Uses two detection methods:

- **Negation patterns**: Scans for phrases like "not X", "no longer X", "instead of X", "changed from X" where X overlaps with existing memory content
- **Value override**: Identifies "subject is/was value" claims where the same subject has different values in old vs new content

#### 2. Staleness

Applies exponential decay based on how many steps have passed since an item was last accessed. The decay follows the formula: `1.0 - exp(-decay_rate * age)`, where the decay rate is derived from a configurable half-life (default 200 steps). Items with staleness above 0.95 are marked for forgetting.

#### 3. Error Poisoning

Tracks items that are consistently present during failures. When an item has been active during a configurable number of consecutive failures (default `poison_threshold=3`), it is flagged for importance reduction. Success resets the failure counter by decrementing it.

#### 4. Redundancy

Detects near-duplicate memories using MD5 content hashing. Items are sorted by importance - when two items hash identically, the lower-importance copy is marked as redundant. The `detect_redundancy()` method also provides word-level Jaccard similarity for finer-grained comparison.

#### 5. Goal Divergence

Identifies items whose content has drifted too far from the current goal. Uses word-level overlap between the item content and goal text. Items below the divergence threshold (default 0.05) that are older than `min_age` steps (default 100) are marked for removal.

### Forgetting Events

Every forgetting action produces a `ForgettingEvent` recording:

- The item ID, trigger type, and reason
- Old and new importance values
- The step number and timestamp
- What item superseded it (for contradiction)
- Whether the action is reversible

## Key Features

- **Five distinct forgetting triggers** covering all major memory quality issues
- **Full provenance** - Every forgetting event is logged with reason, trigger, and reversibility
- **Configurable parameters** - Half-life, poison threshold, redundancy threshold, divergence threshold
- **Graduated response** - Reduces importance rather than immediately deleting, allowing recovery
- **Success/failure tracking** via `record_failure()` and `record_success()` for poison detection
- **Batch scanning** via `scan_redundancy()` and `scan_goal_divergence()` for periodic cleanup
- **Capacity-limited log** - Forgetting log auto-trims at 1000 entries (keeps most recent 500)

## Integration

The `ActiveForgettingEngine` operates during the `CognitiveContextPipeline`'s evaluation phase. Before assembling context, the pipeline evaluates candidate items through the forgetting engine. Items that trigger forgetting have their importance reduced rather than being removed entirely, allowing them to potentially resurface if conditions change.

The engine coordinates with the `EntanglementGraph` - before forgetting an item, the pipeline should check `eviction_check()` to see if the item has strongly entangled partners. The `StateFileManager`'s error journal feeds the failure tracking mechanism, and the `ContextVersioner` records forgetting-driven changes for causal analysis.

## Usage Example

```python
from corteX.engine.cognitive import (
    ActiveForgettingEngine, ForgettingTrigger
)
from corteX.engine.cognitive.active_forgetting import MemoryItem

engine = ActiveForgettingEngine(
    staleness_half_life=200,
    contradiction_decay=0.05,
    poison_threshold=3,
    redundancy_threshold=0.85,
    divergence_threshold=0.05,
)

# Check for contradiction
item = MemoryItem(
    item_id="db_choice",
    content="The database is PostgreSQL",
    importance=0.8,
)
is_contradicted = engine.detect_contradiction(
    item, "The database is now MySQL"
)  # True

# Compute staleness decay
staleness = engine.compute_staleness(
    MemoryItem(item_id="old", content="...", importance=0.5,
               step_created=10, last_accessed_step=50),
    step_number=300,
)  # High value (near 1.0) since 250 steps since last access

# Track failures for poison detection
engine.record_failure(["item_a", "item_b", "item_c"])
engine.record_failure(["item_a", "item_b"])
engine.record_failure(["item_a"])  # item_a now at 3 failures
# Returns [ForgettingEvent(item_id="item_a", trigger=ERROR_POISONING)]

# Reset on success
engine.record_success(["item_a"])  # Decrements failure count

# Batch scan for redundancy
items = [
    MemoryItem(item_id="v1", content="deploy config", importance=0.9),
    MemoryItem(item_id="v2", content="deploy config", importance=0.3),
]
redundant = engine.scan_redundancy(items)
# v2 marked redundant (lower importance, same content hash)

# Scan for goal divergence
diverged = engine.scan_goal_divergence(
    items, goal="deploy service", step_number=500, min_age=100,
)

# Statistics
stats = engine.get_forgetting_stats()
# {"total_forgotten": 3, "by_trigger": {"redundancy": 1, ...},
#  "active_poison_watches": 2}
```

## See Also

- [Entanglement Graph](entanglement.md) - Prevents forgetting items with strong entangled partners
- [Memory Crystallizer](crystallizer.md) - Preserves successful patterns before forgetting details
- [Context Versioner](versioner.md) - Records context state changes caused by forgetting
- [State File Manager](state-file-manager.md) - Error journal feeds failure tracking
