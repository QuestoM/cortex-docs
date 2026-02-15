# Bayesian Mathematics

`corteX.engine.bayesian`

Principled probabilistic foundations for the corteX weight engine. Replaces heuristic EMA updates with conjugate prior distributions, information-theoretic surprise, Kahneman-Tversky prospect theory, and bandit algorithms for exploration vs exploitation.

References: Kahneman & Tversky (1979), Thompson (1933), Auer et al. (2002), Itti & Baldi (2009), Friston (2010).

---

## BetaDistribution

Conjugate prior for binary outcome tracking (success/failure). The posterior updates in closed form: `Beta(alpha + successes, beta + failures)`.

### Constructor

```python
BetaDistribution(alpha: float = 1.0, beta: float = 1.0)
```

A flat `Beta(1, 1)` prior represents complete uncertainty (uniform over `[0, 1]`).

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `mean` | `float` | Posterior mean = `alpha / (alpha + beta)` |
| `variance` | `float` | Posterior variance |
| `std` | `float` | Posterior standard deviation |
| `confidence_interval_95` | `Tuple[float, float]` | Approximate 95% credible interval |
| `total_observations` | `float` | Real observations (excluding prior pseudo-counts) |
| `concentration` | `float` | `alpha + beta` -- higher = more confident |

### Methods

#### `update(success: bool) -> None`

Updates posterior with a single binary observation.

#### `batch_update(successes: int, failures: int) -> None`

Updates with multiple observations at once.

#### `sample() -> float`

Draws a single sample from the Beta posterior (used for Thompson Sampling).

#### `sample_n(n: int) -> List[float]`

Draws n samples from the posterior.

#### `decay(factor: float = 0.99) -> None`

Applies temporal decay by scaling both parameters, increasing uncertainty over time. Used for non-stationary environments.

#### `kl_divergence_from(prior: BetaDistribution) -> float`

Computes KL divergence `D_KL(self || prior)` -- the Bayesian surprise.

#### `to_dict() / from_dict(data)`

Serialization support.

---

## GammaDistribution

Conjugate prior for latency/duration modeling. Posterior updates: `Gamma(shape + n, rate + sum_of_observations)`.

### Constructor

```python
GammaDistribution(shape: float = 2.0, rate: float = 0.002)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `mean` | `float` | Expected latency in ms = `shape / rate` |
| `variance` | `float` | `shape / rate^2` |
| `std` | `float` | Standard deviation |

### Methods

#### `update(observed_latency_ms: float) -> None`

Updates posterior with an observed latency value.

#### `sample() -> float`

Samples a latency value from the posterior.

#### `predictive_surprise(observed_ms: float) -> float`

Returns how surprising an observation is (`0.0` = expected, `1.0` = very surprising).

#### `decay(factor: float = 0.99) -> None`

Temporal decay for non-stationary latencies.

---

## NormalNormalUpdater

Conjugate pair for continuous quality score tracking. Works in precision space (tau = 1/variance) for clean updates.

### Constructor

```python
NormalNormalUpdater(
    prior_mean: float = 0.5,
    prior_precision: float = 1.0,
    known_noise_precision: float = 4.0,
)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `mean` | `float` | Posterior mean |
| `variance` | `float` | `1 / precision` |
| `std` | `float` | Standard deviation |
| `confidence_interval_95` | `Tuple[float, float]` | 95% credible interval |

### Methods

#### `update(observed_quality: float) -> None`

Updates posterior with a quality observation using the conjugate update rule.

#### `sample() -> float`

Samples from the posterior.

#### `kl_divergence_from(prior_mu: float, prior_var: float) -> float`

KL divergence from a prior Normal distribution.

#### `decay(factor: float = 0.99) -> None`

Reduces precision (increases uncertainty) over time.

---

## DirichletMultinomialUpdater

Conjugate pair for categorical choice modeling. Posterior: `Dirichlet(alpha_1 + c_1, ..., alpha_K + c_K)`.

### Constructor

```python
DirichletMultinomialUpdater(
    categories: List[str],
    prior_counts: Optional[List[float]] = None,
)
```

### Methods

#### `update(observed_category: str) -> None`

Updates posterior with an observed category. New categories are added dynamically.

#### `get_probabilities() -> Dict[str, float]`

Returns expected probability for each category (posterior mean).

#### `sample() -> Dict[str, float]`

Samples a probability vector from the Dirichlet posterior.

#### `most_likely() -> str`

Returns the category with the highest expected probability.

#### `entropy() -> float`

Shannon entropy of the expected distribution (lower = more concentrated).

---

## BayesianSurpriseCalculator

Computes Bayesian surprise as KL divergence between prior and posterior. Replaces heuristic surprise in PredictionEngine with principled information-theoretic measurement.

### Static Methods

#### `beta_kl_divergence(post_alpha, post_beta, prior_alpha, prior_beta) -> float`

KL divergence between two Beta distributions.

#### `normal_kl_divergence(post_mu, post_var, prior_mu, prior_var) -> float`

KL divergence between two Normal distributions.

#### `surprise_to_learning_signal(surprise: float, max_surprise: float = 5.0) -> float`

Converts raw KL divergence to a bounded `[0, 1]` learning signal using tanh.

### Instance Methods

#### `compute_beta_surprise(prior: BetaDistribution, posterior: BetaDistribution) -> float`

Bayesian surprise for a Beta distribution update.

#### `compute_normal_surprise(prior_mu, prior_var, post_mu, post_var) -> float`

Bayesian surprise for a Normal distribution update.

---

## ProspectTheoreticUpdater

Kahneman-Tversky Prospect Theory applied to weight updates. Failures are weighted 2.25x more than successes (loss aversion), with diminishing sensitivity for extreme values.

### Constructor

```python
ProspectTheoreticUpdater(
    loss_aversion: float = 2.25,
    gain_sensitivity: float = 0.88,
    loss_sensitivity: float = 0.88,
    probability_gamma: float = 0.61,
)
```

### Methods

#### `value_function(x: float) -> float`

Prospect theory value function. Gains: `x^0.88`. Losses: `-2.25 * |x|^0.88`.

#### `probability_weight(p: float) -> float`

Probability weighting function that overweights small probabilities and underweights large ones. Explains why agents should pay outsized attention to rare failures.

#### `asymmetric_update(current_weight: float, success: bool, learning_rate: float = 0.08, reference_point: Optional[float] = None) -> float`

Updates a weight with loss-aversion asymmetry. A single failure has approximately 2.25x the impact of a single success.

#### `compute_update_delta(outcome_quality: float, expected_quality: float, learning_rate: float = 0.08) -> float`

Computes the prospect-theoretic delta for a continuous outcome relative to a reference point.

---

## BayesianToolSelector

Thompson Sampling for tool/model selection. Maintains a Beta posterior and Gamma latency posterior per tool. Selection: sample from each posterior, pick the highest.

### Methods

#### `register(name: str, prior_alpha: float = 1.0, prior_beta: float = 1.0) -> None`

Registers a tool with an optional informative prior.

#### `record(name: str, success: bool, latency_ms: Optional[float] = None) -> None`

Updates posterior after observing an outcome.

#### `select(candidates: List[str]) -> str`

Thompson Sampling selection: samples from each candidate's Beta posterior, picks the highest.

#### `select_with_latency(candidates: List[str], speed_weight: float = 0.3) -> str`

Thompson Sampling with latency consideration. Combined score: `(1-w)*quality + w*speed`.

#### `get_posterior_summary(name: str) -> Dict[str, Any]`

Returns: `mean`, `std`, `ci_95`, `observations`, `alpha`, `beta`, `latency_mean_ms`, `latency_std_ms`.

#### `decay_all(factor: float = 0.99) -> None`

Applies temporal decay to all posteriors.

---

## UCB1Selector

Upper Confidence Bound algorithm for deterministic selection. Use instead of Thompson Sampling when reproducibility or provable regret bounds are required.

Formula: `UCB1(arm) = avg_reward + sqrt(c * ln(t) / n)`

### Constructor

```python
UCB1Selector(exploration_weight: float = 2.0)
```

### Methods

#### `register(arm: str) -> None`

Registers a new arm.

#### `record(arm: str, reward: float) -> None`

Records a reward observation.

#### `select(candidates: List[str]) -> str`

Selects the arm with the highest UCB1 score. Unplayed arms are always selected first.

---

## AnchorManager

Manages initial weight anchors using historical priors. Addresses Kahneman's anchoring bias: hardcoded `0.5` initialization dominates early behavior. Informed anchors provide better initialization.

### Methods

#### `set_anchor(key: str, value: float, confidence: float = 0.5) -> None`

Sets an informed anchor from historical data or global weights.

#### `get_initial_weight(key: str) -> Tuple[float, float]`

Returns `(weight, confidence)` for a new entity. Defaults to `(0.5, 0.1)` for unknown keys.

#### `debiasing_rate(confidence: float) -> float`

Computes learning rate accounting for anchor confidence. High confidence = lower rate (harder to move).

#### `get_beta_prior(key: str) -> Tuple[float, float]`

Converts an anchor into Beta distribution `(alpha, beta)` parameters. Higher confidence = more concentrated prior (range: 2 to 22 pseudo-observations).

---

## AvailabilityFilter

Controls recency bias in weight updates using a dual-window approach. Detects when recent events genuinely differ from baseline vs. being statistical noise.

### Constructor

```python
AvailabilityFilter(short_window: int = 5, long_window: int = 50)
```

### Methods

#### `record(key: str, success: bool) -> None`

Records an outcome event.

#### `get_adjusted_rate(key: str) -> Tuple[float, bool]`

Returns `(adjusted_success_rate, is_anomalous)`. When short-term rate significantly differs from long-term (deviation > 0.3), flags as anomalous and trusts recent data more (70/30 split).

#### `get_volatility(key: str) -> float`

Computes volatility of recent outcomes (`0.0` = stable, `1.0` = chaotic).

---

## FrameNormalizer

Normalizes decision framing to prevent framing-induced biases. "90% success rate" feels different from "10% failure rate" despite being identical.

### Static Methods

#### `normalize_scores(scores: Dict[str, float]) -> Dict[str, float]`

Maps scores to relative `[0, 1]` range, preventing anchoring from absolute values.

#### `loss_frame_quality(success_rate: float, loss_aversion: float = 2.25) -> float`

Applies loss-frame perspective: `perceived = success_rate - lambda * failure_rate` (normalized to `[0, 1]`).

#### `comparative_frame(scores: Dict[str, float]) -> Dict[str, str]`

Generates human-readable comparative framing (e.g., "best (95.0%)", "12% below best (83.0%)").

### Example

```python
from corteX.engine.bayesian import (
    BetaDistribution,
    BayesianToolSelector,
    ProspectTheoreticUpdater,
    AnchorManager,
)

# Beta posterior tracking
posterior = BetaDistribution(alpha=1.0, beta=1.0)
posterior.update(success=True)
posterior.update(success=True)
posterior.update(success=False)
print(f"Success rate: {posterior.mean:.2f} +/- {posterior.std:.2f}")
# Success rate: 0.60 +/- 0.19

# Thompson Sampling tool selection
selector = BayesianToolSelector()
selector.record("fast_tool", success=True, latency_ms=500)
selector.record("slow_tool", success=True, latency_ms=3000)
best = selector.select_with_latency(["fast_tool", "slow_tool"], speed_weight=0.5)

# Prospect-theoretic updates
prospect = ProspectTheoreticUpdater()
# Failure hits 2.25x harder than success
new_weight = prospect.asymmetric_update(current_weight=0.5, success=False)
print(f"After failure: {new_weight:.3f}")  # drops more than +success would rise
```
