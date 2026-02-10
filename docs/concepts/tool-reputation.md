# Tool Trust and Reputation

corteX applies game-theoretic principles to build a trust system for tools and models. Rather than treating all tools equally, the system tracks reputation over time, fairly attributes credit across multi-tool interactions, and ensures that capability reporting remains honest through incentive-compatible scoring.

## What It Does

The tool reputation system provides three capabilities:

1. **Reputation Tracking**: Modified Tit-for-Tat trust dynamics with quarantine for unreliable tools
2. **Credit Assignment**: Shapley value computation to fairly attribute outcomes across tool coalitions
3. **Truthful Scoring**: VCG-inspired mechanism that incentivizes honest capability reporting

## Why: The Game Theory Inspiration

!!! note "Game Theory: Iterated Games and Mechanism Design"
    The tool reputation system draws on three pillars of game theory:

    **Axelrod's Tournament (1984)**: Robert Axelrod's famous computer tournament showed that Tit-for-Tat -- start cooperative, then mirror your opponent's last move -- outperforms more complex strategies in iterated Prisoner's Dilemma games. The ReputationSystem implements a modified Tit-for-Tat: trust tracks recent behavior, but consecutive failures trigger a "grim trigger" quarantine. This balances forgiveness (EMA smoothing) with protection (exponential quarantine).

    **Shapley Values (1953)**: Lloyd Shapley proved that there is exactly one way to fairly distribute value among players in a cooperative game that satisfies efficiency, symmetry, additivity, and the dummy player property. When multiple tools contribute to an outcome, the ShapleyAttributor computes each tool's marginal contribution -- answering "how much did each tool actually help?"

    **Mechanism Design / VCG (Vickrey-Clarke-Groves)**: In mechanism design, the goal is to create rules that incentivize truthful behavior. The TruthfulScoringMechanism adjusts tool scores by credibility -- the gap between declared capabilities and observed performance. Tools that honestly report their capabilities keep their scores; tools that exaggerate are penalized.

!!! note "Brain Science: Trust and Fear Conditioning"
    The ReputationSystem mirrors the amygdala's role in fear conditioning: repeated negative experiences with a stimulus (tool failures) build an association that suppresses future engagement (quarantine). Recovery follows hippocampal-dependent extinction -- gradual trust rebuilding from a low base, not immediate restoration.

## How It Works

### Reputation System

The `ReputationSystem` tracks trust through iterated interactions using EMA (exponential moving average) with consistency bonuses and grim-trigger quarantine:

```python
from corteX.engine.game_theory import ReputationSystem

reputation = ReputationSystem(
    trust_alpha=0.1,           # EMA learning rate
    consistency_beta=0.05,     # Consistency bonus rate
    quarantine_threshold=3,    # Consecutive failures before quarantine
    quarantine_base_seconds=60.0,  # Base quarantine duration
)

# Record outcomes
reputation.record("code_interpreter", success=True)   # Trust: 0.55
reputation.record("code_interpreter", success=True)   # Trust: 0.60
reputation.record("web_search", success=False)         # Trust: 0.45
reputation.record("web_search", success=False)         # Trust: 0.41
reputation.record("web_search", success=False)         # Trust: 0.37 -> QUARANTINED

# Check trust levels
trust = reputation.get_trust("code_interpreter")  # 0.60
trust = reputation.get_trust("web_search")         # 0.0 (quarantined)
```

#### Trust Evolution

Trust updates follow a three-part formula:

```
trust(t+1) = trust(t) + alpha * (outcome - trust(t))     # EMA update
             + beta * (consistency - 0.5)                  # Consistency bonus
             - penalty(consecutive_failures >= threshold)  # Grim trigger
```

- **EMA update**: Smoothly tracks recent success rate. `alpha = 0.1` means recent outcomes matter more than distant history.
- **Consistency bonus**: Tools with stable behavior (low variance in recent outcomes) receive a bonus. Erratic tools are penalized. Consistency is computed as `1.0 - variance * 4` over the last 20 interactions.
- **Grim trigger**: After `quarantine_threshold` (default: 3) consecutive failures, the tool is quarantined with exponentially increasing duration.

#### Quarantine and Recovery

```python
# Quarantine duration doubles with each additional failure
# 3 consecutive failures: 60 seconds
# 4 consecutive failures: 120 seconds
# 5 consecutive failures: 240 seconds
# duration = base_seconds * 2^(failures - threshold)

# After quarantine expires, trust rebuilds from a low base
# new_trust = max(0.2, old_trust * 0.5)
# The tool is NOT fully trusted again -- it must prove itself

# Manual override: forgive and reset to moderate trust
reputation.forgive("web_search")
# Trust reset to 0.3, quarantine removed
```

#### Tool Ranking and Filtering

```python
# Filter out quarantined tools
available = reputation.get_available_tools(
    ["code_interpreter", "web_search", "calculator"]
)
# ["code_interpreter", "calculator"] (web_search quarantined)

# Rank by trust score
ranked = reputation.get_ranked_tools(
    ["code_interpreter", "calculator", "file_reader"]
)
# [("code_interpreter", 0.60), ("calculator", 0.50), ("file_reader", 0.50)]
```

### Shapley Credit Attribution

The `ShapleyAttributor` fairly distributes credit when multiple tools contribute to an outcome:

```python
from corteX.engine.game_theory import ShapleyAttributor

attributor = ShapleyAttributor()

# Record outcome values for tool coalitions
# "What was the outcome quality when these tools were used together?"
attributor.record_coalition_value({"search"}, 0.4)
attributor.record_coalition_value({"calculator"}, 0.3)
attributor.record_coalition_value({"search", "calculator"}, 0.9)
attributor.record_coalition_value(set(), 0.0)

# Compute Shapley values
shapley = attributor.compute({"search", "calculator"})
# {"search": 0.5, "calculator": 0.4}
# search contributed more because it raised coalition value more
```

#### How Shapley Values Work

The Shapley value for tool `i` is the average marginal contribution of `i` across all possible orderings of tools:

```
phi_i = SUM over all coalitions S not containing i:
    [|S|! * (|N|-|S|-1)! / |N|!] * [v(S + {i}) - v(S)]
```

This satisfies four properties that uniquely characterize "fair" attribution:

| Property | Meaning |
|----------|---------|
| **Efficiency** | Credits sum to total value |
| **Symmetry** | Equal contributors get equal credit |
| **Additivity** | Credits across games add up |
| **Dummy** | Non-contributors get zero credit |

#### Exact vs. Approximate Computation

```python
# For N <= 8 tools: exact computation (O(2^N * N))
shapley = attributor.compute_exact({"a", "b", "c"})

# For N > 8 tools: Monte Carlo approximation
shapley = attributor.compute_approximate(
    {"tool_1", "tool_2", ..., "tool_12"},
    num_permutations=100,
)

# Auto-selection based on player count
shapley = attributor.compute(players)  # Picks exact or approximate
```

#### Credit Allocation

Distribute a concrete reward proportionally to Shapley values:

```python
allocation = attributor.get_credit_allocation(
    all_players={"search", "calculator", "code_runner"},
    total_reward=1.0,
)
# {"search": 0.45, "calculator": 0.35, "code_runner": 0.20}
# Credits sum to exactly 1.0
```

#### Incremental Tracking

For real-time updates without full recomputation:

```python
# Update running Shapley estimate incrementally
attributor.update_running("search", marginal_contribution=0.3, alpha=0.1)
attributor.update_running("calculator", marginal_contribution=0.2, alpha=0.1)

# Query running estimates
estimates = attributor.get_running_shapley()
# {"search": 0.03, "calculator": 0.02}
```

### Truthful Scoring Mechanism

The `TruthfulScoringMechanism` ensures tools cannot game the system by exaggerating their capabilities:

```python
from corteX.engine.game_theory import TruthfulScoringMechanism

scorer = TruthfulScoringMechanism()

# Tool declares its capabilities
scorer.declare("fast_search", {
    "speed": 0.95,
    "accuracy": 0.80,
    "coverage": 0.70,
})

# System observes actual performance
scorer.observe("fast_search", {"speed": 0.90, "accuracy": 0.60})
scorer.observe("fast_search", {"speed": 0.88, "accuracy": 0.55})
# Over time, observations converge via EMA (alpha=0.15)
```

#### Credibility Scoring

Credibility measures how well declared capabilities match observed performance:

```python
credibility = scorer.credibility_score("fast_search")
# Computes: 1.0 - average_error * 2
# If declared accuracy=0.80 but observed=0.55, error=0.25
# If declared speed=0.95 but observed=0.89, error=0.06
# Average error = 0.155, credibility = 1.0 - 0.31 = 0.69
```

| Credibility | Interpretation |
|-------------|---------------|
| 1.0 | Perfectly honest -- declared matches observed exactly |
| 0.7-0.9 | Reasonably honest -- minor discrepancies |
| 0.4-0.7 | Suspect -- significant gaps between claims and reality |
| 0.0-0.4 | Dishonest -- declared capabilities far exceed actual performance |
| 0.5 | Unknown -- not enough observations yet (neutral default) |

#### Score Adjustment

Raw tool scores are multiplied by credibility, so honest tools keep their scores while exaggerators are penalized:

```python
# Honest tool (credibility = 0.95)
adjusted = scorer.adjusted_score("honest_tool", raw_score=0.8)
# 0.8 * 0.95 = 0.76

# Exaggerating tool (credibility = 0.4)
adjusted = scorer.adjusted_score("liar_tool", raw_score=0.8)
# 0.8 * 0.4 = 0.32
```

This creates an incentive structure where tools benefit from truthful self-reporting: exaggerating capabilities reduces credibility, which reduces the adjusted score, making the tool less likely to be selected.

#### Monitoring All Tools

```python
# Get credibility scores for all known tools
all_cred = scorer.get_all_credibilities()
# {"fast_search": 0.69, "calculator": 0.95, "code_runner": 0.82}
```

### Integration: How the Three Systems Work Together

The reputation system, Shapley attribution, and truthful scoring form a reinforcing cycle:

1. **Selection**: ReputationSystem filters out quarantined tools and ranks candidates by trust
2. **Execution**: Multiple tools may contribute to a task outcome
3. **Attribution**: ShapleyAttributor computes each tool's marginal contribution
4. **Scoring**: TruthfulScoringMechanism adjusts scores by credibility
5. **Update**: ReputationSystem records success/failure, updating trust for next iteration

```python
# 1. Get available tools (filters quarantined)
available = reputation.get_available_tools(candidates)

# 2. After task execution with multiple tools...
attributor.record_coalition_value({"search", "code"}, quality)
attributor.record_coalition_value({"search"}, search_only_quality)

# 3. Compute attribution
credit = attributor.get_credit_allocation(
    {"search", "code"}, total_reward=quality
)

# 4. Adjust by credibility
for tool, raw_credit in credit.items():
    adjusted = scorer.adjusted_score(tool, raw_credit)
    reputation.record(tool, success=(adjusted > 0.5))
```

## When It Activates

- **Before tool selection**: ReputationSystem filters quarantined tools and ranks by trust
- **After task completion**: Outcomes are recorded, trust scores updated
- **During multi-tool tasks**: ShapleyAttributor tracks coalition values for credit assignment
- **At consolidation**: TruthfulScoringMechanism compares declared vs. observed performance
- **On consecutive failures**: Grim trigger activates quarantine with exponential duration
- **On quarantine expiry**: Trust rebuilds from a low base (not full restoration)

## API Reference

```python
from corteX.engine.game_theory import (
    ReputationSystem,
    ShapleyAttributor,
    TruthfulScoringMechanism,
    MinimaxSafetyGuard,
)
```
