# NeuroLlama API Reference

## Package: `corteX.neurollama`

Neuroscience-enhanced transformer architecture. Augments standard Llama-style attention with synaptic modulation, cortical columns, predictive coding, Hebbian learning, habituation, population coding, and early exit mechanisms. All neuroscience modifications are independently toggleable and operate on NumPy arrays.

---

## config

Central configuration dataclass parameterizing the standard Llama architecture and every neuroscience-inspired modification.

### `NeuroLlamaConfig`

**Type**: `@dataclass`

Complete configuration for a NeuroLlama model covering architecture, neuroscience toggles, and training objective weights.

#### Architecture Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `num_layers` | `int` | `32` | Number of transformer layers |
| `hidden_dim` | `int` | `4096` | Hidden dimension |
| `num_heads` | `int` | `32` | Number of query attention heads |
| `num_kv_heads` | `int` | `8` | Number of key-value heads (GQA) |
| `head_dim` | `int` | `128` | Dimension per attention head |
| `ffn_dim` | `int` | `14336` | Feed-forward network intermediate dimension |
| `vocab_size` | `int` | `128256` | Vocabulary size |
| `max_seq_len` | `int` | `4096` | Maximum sequence length |
| `rope_theta` | `float` | `500000.0` | RoPE frequency base |

#### Neuroscience Toggle Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `enable_synaptic_modulation` | `bool` | `True` | Enable per-head synaptic scaling |
| `synaptic_mode` | `str` | `"static"` | Modulation tier: `"static"`, `"context_dependent"`, `"full_matrix"` |
| `enable_cortical_columns` | `bool` | `True` | Enable GQA groups as cortical columns |
| `enable_lateral_inhibition` | `bool` | `True` | Enable cross-column inhibition |
| `enable_predictive_coding` | `bool` | `True` | Enable top-down predictive coding |
| `enable_hebbian_attention` | `bool` | `True` | Enable within-sequence Hebbian learning |
| `hebbian_lr` | `float` | `0.01` | Hebbian learning rate |
| `enable_habituation` | `bool` | `True` | Enable stimulus-specific adaptation |
| `habituation_rate` | `float` | `0.1` | Habituation suppression rate |
| `recovery_rate` | `float` | `0.01` | Habituation recovery rate |
| `enable_population_coding` | `bool` | `True` | Enable population-coded output |
| `enable_early_exit` | `bool` | `True` | Enable System 1/2 early exit |
| `exit_layers` | `List[int]` | `[8, 16, 24]` | Layers with exit classifiers |
| `exit_confidence_threshold` | `float` | `0.9` | Confidence required for early exit |

#### Training Weight Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `prediction_error_weight` | `float` | `0.1` | Weight for prediction error loss |
| `specialization_weight` | `float` | `0.01` | Weight for column specialization loss |
| `calibration_weight` | `float` | `0.01` | Weight for calibration loss |
| `efficiency_weight` | `float` | `0.05` | Weight for metabolic efficiency loss |
| `coherence_weight` | `float` | `0.05` | Weight for goal coherence loss |

#### Derived Properties

| Property | Type | Description |
|----------|------|-------------|
| `gqa_ratio` | `int` | Query heads per KV head (`num_heads // num_kv_heads`) |
| `num_groups` | `int` | Number of GQA groups (= cortical columns) |
| `heads_per_group` | `int` | Query heads within each GQA group |

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_llama_config` | `(llama_config, enable_neuro=True) -> NeuroLlamaConfig` | Create from HuggingFace Llama config. Computes exit layers at 25/50/75% depth. |
| `to_dict` | `() -> Dict[str, Any]` | Serialize to JSON-safe dictionary |
| `from_dict` | `(data) -> NeuroLlamaConfig` | Deserialize from dictionary (unknown keys silently dropped) |
| `to_json` | `(indent=2) -> str` | Serialize to JSON string |
| `from_json` | `(json_str) -> NeuroLlamaConfig` | Deserialize from JSON string |
| `with_overrides` | `(**overrides) -> NeuroLlamaConfig` | Return new config with selected fields overridden |

#### Validation

Post-init validation enforces:

- `synaptic_mode` must be one of `{"static", "context_dependent", "full_matrix"}`
- `num_heads` must be divisible by `num_kv_heads`
- `hidden_dim`, `head_dim`, `num_layers` must be positive
- `exit_confidence_threshold` must be in `(0, 1]`
- All `exit_layers` indices must be within `[0, num_layers)`

#### Example

```python
from corteX.neurollama.config import NeuroLlamaConfig

# Default 8B config
config = NeuroLlamaConfig()

# Custom config with selective neuro features
config = NeuroLlamaConfig(
    num_layers=16,
    hidden_dim=2048,
    num_heads=16,
    num_kv_heads=4,
    enable_hebbian_attention=False,
    enable_early_exit=True,
    exit_layers=[4, 8, 12],
)

# Override from existing config
fast_config = config.with_overrides(enable_early_exit=True, exit_confidence_threshold=0.85)
```

---

## synaptic_attention

Three tiers of biologically-inspired attention modulation.

### `SynapticScaling`

Per-head learnable scaling factors (simplest tier). Each head receives a scalar alpha that amplifies or attenuates its contribution.

```python
SynapticScaling(num_heads: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(attention_scores: ndarray) -> ndarray` | Scale each head's scores. Input: `(batch, num_heads, seq_q, seq_k)`. |
| `update_alpha` | `(head_idx: int, delta: float) -> None` | Adjust a specific head's synaptic strength |
| `get_alphas` | `() -> ndarray` | Copy of current alpha values `(num_heads,)` |

### `NeuromodulatedAttention`

Context-dependent gating mimicking neuromodulators (dopamine, serotonin). A two-layer network maps a context vector to per-head gating factors in `[0, 1]`.

```python
NeuromodulatedAttention(hidden_dim: int, num_heads: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_modulation` | `(context: ndarray) -> ndarray` | Per-head gates `(batch, num_heads)` from context `(batch, hidden_dim)` |
| `forward` | `(attention_output: ndarray, context: ndarray) -> ndarray` | Apply gating to attention output `(batch, num_heads, seq, head_dim)` |

### `SynapticModulationMatrix`

Full pairwise query-key modulation within a local attention window. Richest and most expensive tier.

```python
SynapticModulationMatrix(head_dim: int, window_size: int = 512)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_synaptic_strength` | `(query, key) -> ndarray` | Pairwise strength `(..., seq_q, seq_k)` in `[0, 1]` |
| `forward` | `(attention_scores, query, key) -> ndarray` | Element-wise modulate scores by synaptic strength |

### `scaled_dot_product_attention`

```python
def scaled_dot_product_attention(Q, K, V, mask=None) -> ndarray
```

Standard scaled dot-product attention baseline implemented in NumPy.

---

## cortical_columns

GQA key-value groups as functional cortical columns with lateral inhibition and hierarchical organization.

### `CorticalColumnAttention`

Each KV group is an independent "column" processing its share of query heads. Computes a JS divergence metric quantifying specialization.

```python
CorticalColumnAttention(num_groups: int, heads_per_group: int, head_dim: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_group_attention` | `(Q_group, K_group, V_group, group_idx, mask) -> (output, weights)` | Attention for a single column |
| `compute_specialization_divergence` | `(group_attentions: List[ndarray]) -> float` | Mean pairwise JS divergence (higher = more specialized) |
| `forward` | `(Q, K, V, mask) -> (output, specialization_div)` | Full cortical-column attention |

### `LateralInhibition`

Cross-column inhibitory connections preventing redundant representations. Uses anti-Hebbian learning.

```python
LateralInhibition(num_groups: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(group_outputs: List[ndarray]) -> List[ndarray]` | Apply lateral inhibition across columns |
| `update_inhibition` | `(group_activations: ndarray, lr=0.01) -> None` | Anti-Hebbian update: co-active columns develop mutual inhibition |
| `get_inhibition_matrix` | `() -> ndarray` | Copy of current inhibition matrix |

### `HierarchicalColumnOrganizer`

Assigns roles to columns by layer depth, mirroring the ventral visual hierarchy: syntactic (layers 0-9), semantic (layers 10-19), abstract (layers 20+).

```python
HierarchicalColumnOrganizer(num_layers: int, num_groups: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `get_column_role` | `(layer_idx, group_idx) -> str` | Returns `"syntactic"`, `"semantic"`, or `"abstract"` |
| `get_specialization_targets` | `(layer_idx) -> Dict` | Target distribution and diversity target for a layer |

---

## predictive_coding

Higher layers predict lower-layer representations. Prediction errors drive learning and quantify surprise.

### `PredictiveCodingConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `prediction_weight` | `float` | `0.1` | Weight of prediction error correction |
| `contrastive_weight` | `float` | `0.05` | Weight for CPC loss |
| `num_negative_samples` | `int` | `16` | Negative samples for InfoNCE |
| `prediction_horizon` | `int` | `1` | How far ahead to predict |
| `error_signal_decay` | `float` | `0.9` | Exponential decay for accumulated surprise |

### `PredictionHead`

Per-layer linear projection that predicts the next layer's input.

```python
PredictionHead(hidden_dim: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `predict` | `(current_layer_output) -> ndarray` | Generate predicted representation |
| `compute_error` | `(predicted, actual) -> ndarray` | Element-wise squared prediction error |

### `PredictiveCodingLayer`

Transformer layer wrapper with top-down predictive coding.

```python
PredictiveCodingLayer(hidden_dim: int, config: Optional[PredictiveCodingConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(x, prediction_from_above) -> (output, prediction_for_below, surprise)` | Forward pass with predictive coding |
| `get_accumulated_surprise` | `() -> float` | Total decayed surprise across sequence |
| `reset` | `() -> None` | Reset surprise for new sequence |

### `ContrastivePredictiveCoding`

CPC / InfoNCE loss for self-supervised representation learning.

```python
ContrastivePredictiveCoding(hidden_dim: int, num_negatives: int = 16, prediction_horizon: int = 1)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `score` | `(context, target) -> ndarray` | Bilinear score: `h^T @ W_cpc @ x` |
| `sample_negatives` | `(embeddings, num_negatives) -> ndarray` | Random negative samples from batch |
| `compute_loss` | `(hidden_states, target_embeddings) -> float` | InfoNCE loss over batch |

### `PredictionErrorSignal`

Manages prediction error signals across the full model.

```python
PredictionErrorSignal(num_layers: int, hidden_dim: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `collect_errors` | `(layer_idx, error) -> None` | Accumulate error for a layer |
| `get_total_loss` | `() -> float` | Sum of prediction error across all layers |
| `get_surprise_profile` | `() -> ndarray` | Per-layer surprise values `(num_layers,)` |
| `reset` | `() -> None` | Clear all accumulated errors |
| `get_stats` | `() -> Dict` | Mean/max surprise, error distribution |

---

## hebbian_attention

Within-sequence Hebbian learning for attention, reward-modulated plasticity, and fast-weight FFN layers.

### `HebbianAttention`

Attention with within-sequence Hebbian learning. Maintains a co-activation matrix H that biases future attention: `A = softmax(QK^T/sqrt(d) + lr * QHK^T)`.

```python
HebbianAttention(head_dim: int, config: Optional[HebbianConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(Q, K, V, reset=False) -> ndarray` | Hebbian-modulated attention. Input: `(batch, seq_len, head_dim)`. |
| `reset` | `() -> None` | Zero out H |
| `get_hebbian_strength` | `() -> float` | Frobenius norm of H |

### `RewardModulatedHebbian`

Three-factor learning: pre x post x reward (STDP analogue). Dopaminergic reward signals gate plasticity.

```python
RewardModulatedHebbian(head_dim: int, config: Optional[HebbianConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(Q, K, V, reward=None) -> ndarray` | Reward-modulated attention |
| `set_reward` | `(reward: float) -> None` | Set reward signal for next update |
| `get_stats` | `() -> Dict` | `hebbian_strength`, `current_reward`, `step_count`, `mean_H`, `max_H` |
| `reset` | `() -> None` | Zero out H and reward |

### `HebbianFastWeights`

Hebbian plasticity applied to FFN weights for rapid binding / short-term memory: `y = (W_base + fast_W) @ x`.

```python
HebbianFastWeights(input_dim: int, output_dim: int, config: Optional[HebbianConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(x) -> ndarray` | Compute output using base + fast weights |
| `accumulate` | `(input_activation, output_activation) -> None` | Hebbian update to fast weights |
| `decay` | `() -> None` | Apply exponential decay to fast weights |
| `reset` | `() -> None` | Zero out fast weights |
| `get_fast_weight_strength` | `() -> float` | Frobenius norm of fast-weight matrix |
| `get_stats` | `() -> Dict` | Norm ratio, step count, mean/max values |

---

## habituation_layer

Stimulus-specific adaptation (SSA) for attention.

### `HabituatingAttention`

Per-head habituation suppressing repeated attention patterns with pattern counting and recovery.

```python
HabituatingAttention(num_heads: int, max_seq_len: int, config: Optional[HabituationConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(attention_scores, seq_len) -> ndarray` | Apply habituation and renormalize. Input: `(batch, num_heads, seq_len, seq_len)`. |
| `reset` | `() -> None` | Zero all pattern counts |
| `get_habituation_map` | `() -> ndarray` | Current suppression map `(num_heads, max_seq_len, max_seq_len)` |

### `AttentionDecay`

Cross-layer cumulative decay preventing fixation on specific tokens.

```python
AttentionDecay(decay_rate: float = 0.05)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(attention_scores, cumulative_attention=None) -> (decayed_scores, new_cumulative)` | Apply novelty-based decay and update cumulative state |

### `NoveltyDetector`

Dishabituation boost for surprising tokens.

```python
NoveltyDetector(num_heads: int, surprise_threshold: float = 2.0)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `detect_novelty` | `(current_attention, habituation_state) -> ndarray` | Binary novelty mask `(batch, num_heads, seq_q, seq_k)` |
| `get_novelty_scores` | `() -> ndarray` | Per-position novelty `(num_heads, seq_k)` |
| `get_dishabituation_boost` | `() -> ndarray` | Multiplicative boost `[1.0, 2.0]` for novel tokens |

---

## population_output

Population-coded output aggregation: attention heads "vote" for the final representation via confidence, tuning curves, or entropy.

### `PopulationCodedAttention`

Gaussian tuning-curve population vectors. Each head has a preferred direction in feature space.

```python
PopulationCodedAttention(config: Optional[PopulationConfig] = None)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_activation` | `(head_contexts) -> ndarray` | Gaussian activation `(batch, num_heads)` |
| `forward` | `(head_outputs, head_contexts) -> ndarray` | Population-coded output `(batch, hidden_dim)` |
| `update_preferred_directions` | `(head_outputs, rewards, lr=0.001) -> None` | Hebbian direction update |

### `ConfidenceWeightedVoting`

Learned per-head confidence estimators for weighted voting. Heads below threshold are silenced.

```python
ConfidenceWeightedVoting(num_heads: int, head_dim: int, confidence_threshold: float = 0.1, temperature: float = 1.0)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `estimate_confidence` | `(head_outputs) -> ndarray` | Per-head confidence `(batch, num_heads)` in `[0, 1]` |
| `forward` | `(head_outputs) -> ndarray` | Confidence-weighted vote `(batch, head_dim)` |
| `update_estimators` | `(head_outputs, outcome_quality, lr=0.001) -> None` | Update toward ground-truth quality |

### `EntropyBasedVoting`

Parameter-free entropy-based voting. Low entropy = confident = higher voting weight.

```python
EntropyBasedVoting(num_heads: int, temperature: float = 1.0)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_head_entropy` | `(attention_weights) -> ndarray` | Shannon entropy per head |
| `forward` | `(head_outputs, attention_weights) -> ndarray` | Entropy-weighted aggregation `(batch, head_dim)` |

### `PopulationVectorDecoder`

Project population vector to vocabulary logits (replaces lm_head).

```python
PopulationVectorDecoder(hidden_dim: int, vocab_size: int)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `decode` | `(population_vector) -> ndarray` | Logits `(batch, vocab_size)` |
| `decode_with_confidence` | `(population_vector, head_confidences, top_k=8) -> (logits, confidence)` | Decode with overall confidence from top-k heads |

---

## model

Full NeuroLlama model assembly.

### `RMSNorm`

Root Mean Square layer normalization (Zhang & Sennrich, 2019).

```python
RMSNorm(hidden_dim: int, eps: float = 1e-6)
```

Method: `forward(x) -> ndarray` -- Normalize x by its RMS along the last dimension.

### `RotaryPositionEmbedding`

Rotary Position Embedding (RoPE) for Llama-style models.

```python
RotaryPositionEmbedding(head_dim: int, max_seq_len: int = 4096, theta: float = 500000.0)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_rotary_embedding` | `(seq_len) -> (cos, sin)` | Cos/sin tables clipped to seq_len |
| `apply` | `(x, position_ids) -> ndarray` | Apply rotary embedding to x at given positions |

### `SwiGLU`

SwiGLU feed-forward network (Shazeer, 2020).

```python
SwiGLU(hidden_dim: int, ffn_dim: int)
```

Method: `forward(x) -> ndarray` -- `(gate * sigmoid(gate)) * up @ W_down`.

### `NeuroLlamaBlock`

Single transformer block with all neuroscience modifications: RMSNorm, GQA attention, SwiGLU FFN, predictive coding, and early exit.

```python
NeuroLlamaBlock(layer_idx: int, config: NeuroLlamaConfig)
```

```python
def forward(
    self, x, mask=None, kv_cache=None,
    prediction_from_above=None, cumulative_attention=None, reward=None,
) -> Dict[str, Any]
```

**Returns**: Dict with keys: `output`, `prediction`, `surprise`, `cumulative_attention`, `kv_cache`, `exit_confidence`.

### `NeuroLlamaModel`

Full model: embedding -> N blocks -> norm -> lm_head.

```python
NeuroLlamaModel(config: NeuroLlamaConfig)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `forward` | `(input_ids, mask=None, goal_embedding=None, reward=None) -> Dict` | Full forward pass. Returns `logits`, `exit_layer`, `system_type`, `surprises`, `predictions`, `hidden_states`. |
| `get_stats` | `() -> Dict` | Per-layer surprise, early exit counts, approximate total parameters |
| `reset_sequence_state` | `() -> None` | Reset Hebbian matrices, habituation counts, cumulative attention |

#### Example

```python
from corteX.neurollama.model import create_neurollama
import numpy as np

model = create_neurollama("8B")
result = model.forward(input_ids=np.array([1, 2, 3, 4]))

logits = result["logits"]           # (1, seq_len, vocab_size)
exit_layer = result["exit_layer"]   # which layer exited (or num_layers if full pass)
system_type = result["system_type"] # "system1" or "system2"
surprises = result["surprises"]     # per-layer prediction error

stats = model.get_stats()
# {"per_layer_surprise": [...], "early_exit_counts": {...}, "total_params_approx": ...}

model.reset_sequence_state()
```

### `create_neurollama`

```python
def create_neurollama(config_or_preset: str | NeuroLlamaConfig = "8B") -> NeuroLlamaModel
```

Factory function to create a NeuroLlamaModel from a preset name or a NeuroLlamaConfig.

**Presets**:

| Preset | Layers | Hidden | Heads | KV Heads | FFN | Exit Layers |
|--------|--------|--------|-------|----------|-----|-------------|
| `"8B"` | 32 | 4096 | 32 | 8 | 14336 | [8, 16, 24] |
| `"70B"` | 80 | 8192 | 64 | 8 | 28672 | [20, 40, 60] |
| `"405B"` | 126 | 16384 | 128 | 16 | 53248 | [32, 64, 96] |

---

## training_objectives

Brain-inspired training loss functions for NeuroLlama.

### `TrainingObjectiveConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `prediction_error_weight` | `float` | `0.1` | Surprise loss weight |
| `specialization_weight` | `float` | `0.01` | Column divergence weight |
| `calibration_weight` | `float` | `0.01` | ECE loss weight |
| `efficiency_weight` | `float` | `0.05` | Metabolic cost weight |
| `coherence_weight` | `float` | `0.05` | Goal alignment weight |
| `contrastive_weight` | `float` | `0.05` | CPC loss weight |
| `num_calibration_bins` | `int` | `10` | Number of ECE bins |

### Individual Loss Classes

| Class | Formula | Method |
|-------|---------|--------|
| `SurpriseLoss` | `sum_l \|\|h_l - f_l(h_{l+1})\|\|^2` | `compute(predicted, actual) -> float` |
| `SpecializationLoss` | Negative pairwise JS divergence | `compute(group_attention_distributions) -> float` |
| `CalibrationLoss` | Expected Calibration Error | `compute(confidences, correctness) -> float` |
| `EfficiencyLoss` | Sum of layer usage mask | `compute(layer_usage_mask) -> float` |
| `CoherenceLoss` | Negative cosine similarity | `compute(hidden_states, goal_embedding) -> float` |

### `NeuroCompositeLoss`

Multi-objective weighted combination with phase scheduling.

```python
NeuroCompositeLoss(config: TrainingObjectiveConfig)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute` | `(predictions, targets, attention_dists, confidences, correctness, layer_usage, hidden_states, goal_embedding) -> (total, components)` | Weighted composite loss |
| `get_loss_history` | `() -> Dict[str, List[float]]` | Per-component loss history |
| `adjust_weights` | `(phase: int) -> None` | Phase scheduling: 1=pretrain, 2=continued, 3=finetune |

### `BrainInspiredReward`

RLBF reward shaping: `R = w1*pred + w2*surprise + w3*eff + w4*coh + w5*nov - w6*rep`.

```python
BrainInspiredReward(
    w_pred=0.25, w_surprise=0.15, w_efficiency=0.15,
    w_coherence=0.20, w_novelty=0.15, w_repetition=0.10,
)
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_prediction_accuracy` | `(predictions, actuals) -> float` | Mean cosine similarity |
| `compute_efficiency_score` | `(layers_used, total_layers) -> float` | 1.0 = fewest layers |
| `compute_repetition_penalty` | `(outputs, n=3) -> float` | N-gram repetition penalty `[0, 1]` |
| `compute_reward` | `(trajectory: Dict) -> float` | Composite RLBF reward from trajectory |

#### Example

```python
from corteX.neurollama.training_objectives import NeuroCompositeLoss, TrainingObjectiveConfig

loss_fn = NeuroCompositeLoss(TrainingObjectiveConfig(
    prediction_error_weight=0.1,
    specialization_weight=0.01,
))

total, components = loss_fn.compute(
    predictions, targets, group_attentions,
    confidences, correctness, layer_mask,
    final_hidden, goal_vec,
)

# Phase scheduling
loss_fn.adjust_weights(phase=2)  # continued pretraining
history = loss_fn.get_loss_history()
```

---

## Performance Notes

- All operations use NumPy -- no PyTorch/TensorFlow dependency required
- Weight initialization uses Xavier/He uniform for stable training
- Softmax implementations are numerically stable (max-subtraction)
- Sigmoid implementations are clipped to prevent overflow
- Early exit confidence uses GELU approximation (no scipy dependency)
- Factory presets match Llama 3.1 architecture specifications

---

## See Also

- [Inference Hooks API](../engine/inference-hooks.md) -- Inference-time hooks for existing models (Layer 2)
- [Brain State Injector API](../engine/brain-state-injector.md) -- Brain state compilation for LLM prompts
- [Calibration API](../engine/calibration.md) -- System-level calibration connecting to model confidence
- [Prediction Engine API](../engine/prediction.md) -- Agent-level prediction and surprise
