# Inference Hooks (Layer 2) API Reference

## Modules: `corteX.engine.inference_hooks`, `neuro_adapter`, `training_collector`, `early_exit`

Inference-time brain modifications for local models. These modules modify attention patterns during the forward pass without requiring additional training. The Layer 2 stack spans four modules: inference hooks (attention modification), LoRA adapters (fine-tuning framework), training data collection (SFT/DPO data pipeline), and early exit (System 1/2 dual-process inference).

---

## inference_hooks

Neuroscience-inspired hooks that modify attention patterns and outputs during the forward pass of local models. No model training required.

### Configuration Dataclasses

#### `HebbianConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `decay` | `float` | `0.99` | Exponential decay factor for the co-activation matrix H |
| `modulation_strength` | `float` | `0.1` | Scaling factor for Hebbian bias on attention logits |
| `max_matrix_dim` | `int` | `2048` | Maximum dimension for the co-activation matrix |

#### `HabituationConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `habituation_rate` | `float` | `0.1` | Speed at which repeated attention is suppressed |
| `recovery_rate` | `float` | `0.01` | Speed at which habituation decays (dishabituation) |

#### `TemperatureConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `alpha` | `float` | `0.5` | Sensitivity of temperature to entropy deviation |
| `temp_min` | `float` | `0.5` | Minimum allowable temperature |
| `temp_max` | `float` | `2.0` | Maximum allowable temperature |

#### `VotingConfig`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `suppress_threshold` | `float` | `0.15` | Confidence below which heads are suppressed |
| `suppress_below` | `bool` | `True` | Whether to zero out low-confidence heads |

#### `InferenceHookConfig`

Master configuration combining all hook sub-configurations with enable/disable toggles.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `hebbian` | `HebbianConfig` | `HebbianConfig()` | Hebbian accumulator parameters |
| `habituation` | `HabituationConfig` | `HabituationConfig()` | Habituation parameters |
| `temperature` | `TemperatureConfig` | `TemperatureConfig()` | Temperature scaling parameters |
| `voting` | `VotingConfig` | `VotingConfig()` | Population voting parameters |
| `enable_hebbian` | `bool` | `True` | Enable Hebbian co-activation tracking |
| `enable_habituation` | `bool` | `True` | Enable stimulus-specific adaptation |
| `enable_temperature` | `bool` | `True` | Enable adaptive head temperature |
| `enable_voting` | `bool` | `True` | Enable population-weighted voting |

### `HebbianAccumulator`

Short-term Hebbian co-activation tracker (Eliasmith et al., 2024). Maintains a co-activation matrix H that biases future attention scores toward co-activated positions. Exponential decay prevents runaway potentiation. Resets per sequence.

#### Constructor

```python
HebbianAccumulator(config: Optional[HebbianConfig] = None)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `update` | `(query: np.ndarray, key: np.ndarray, attention_weights: np.ndarray) -> None` | Update H from current attention. Arrays are `(seq_len, d_k\|seq_len)`. |
| `modulate` | `(attention_scores: np.ndarray) -> np.ndarray` | Bias pre-softmax logits `(seq_len, seq_len)` using accumulated H. Returns unmodified scores if H is uninitialized. |
| `reset` | `() -> None` | Reset matrix and update count for a new sequence |
| `get_stats` | `() -> Dict[str, Any]` | Returns `update_count`, `matrix_allocated`, `h_frobenius_norm` |

### `AttentionHabituation`

Stimulus-specific adaptation for repeated attention patterns (Ulanovsky et al., 2003). Tracks cumulative attention per position across layers. Heavily attended tokens receive a novelty penalty; novel tokens trigger dishabituation.

#### Constructor

```python
AttentionHabituation(config: Optional[HabituationConfig] = None)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `apply_habituation` | `(attention_scores: np.ndarray, layer_idx: int) -> np.ndarray` | Reduce attention for heavily-attended tokens. Input: `(seq_len, seq_len)` pre-softmax logits. Returns modified logits. |
| `reset` | `() -> None` | Reset cumulative trackers for a new sequence |
| `get_stats` | `() -> Dict[str, Any]` | Returns `tracked_layers`, `layer_indices` |

### `AdaptiveHeadTemperature`

Per-head temperature scaling based on attention entropy. Low entropy (confident) heads get lower temperature (sharpened), high entropy heads get higher temperature (softened). Formula: `temp_h = 1 + alpha * (H_h - mean_H) / std_H`, clamped to `[temp_min, temp_max]`.

#### Constructor

```python
AdaptiveHeadTemperature(config: Optional[TemperatureConfig] = None)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_temperatures` | `(attention_logits: np.ndarray) -> np.ndarray` | Per-head temperatures from `(num_heads, seq_len, seq_len)` logits. Returns `(num_heads,)` temperatures. |
| `apply_temperatures` | `(attention_logits: np.ndarray) -> np.ndarray` | Divide each head's logits by its adaptive temperature. Input and output: `(num_heads, seq_len, seq_len)`. |

### `PopulationWeightedVoting`

Confidence-weighted aggregation of attention head outputs (Georgopoulos, 1986). Weights each head by confidence (inverse entropy). Uncertain heads are down-weighted; optionally suppresses heads below threshold (lateral inhibition).

#### Constructor

```python
PopulationWeightedVoting(config: Optional[VotingConfig] = None)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_confidence` | `(head_outputs: np.ndarray, attention_weights: np.ndarray) -> np.ndarray` | Per-head confidence in `[0,1]` from `(num_heads, seq_len, seq_len)` weights. Returns `(num_heads,)`. |
| `aggregate` | `(head_outputs: np.ndarray, confidences: np.ndarray) -> np.ndarray` | Weighted voting: `(num_heads, seq_len, d_v)` to `(seq_len, d_v)`. Suppresses heads below threshold if enabled. |

### `InferenceHookPipeline`

Orchestrates all inference-time hooks in sequence: pre_attention (habituation + temperature) -> softmax -> post_attention (Hebbian) -> output_aggregation (population voting). Reset per sequence.

#### Constructor

```python
InferenceHookPipeline(config: Optional[InferenceHookConfig] = None)
```

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `hebbian` | `HebbianAccumulator` | The Hebbian accumulator instance |
| `habituation` | `AttentionHabituation` | The habituation instance |
| `temperature` | `AdaptiveHeadTemperature` | The temperature scaling instance |
| `voting` | `PopulationWeightedVoting` | The voting instance |

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `apply_pre_attention` | `(attention_logits: np.ndarray, layer_idx: int) -> np.ndarray` | Pre-attention: habituation + adaptive temperature. Input: `(num_heads, seq_len, seq_len)`. |
| `apply_post_attention` | `(attention_weights: np.ndarray, query: np.ndarray, key: np.ndarray) -> None` | Post-attention: update Hebbian matrix. All arrays `(num_heads, ...)`. |
| `apply_output_aggregation` | `(head_outputs: np.ndarray, attention_weights: np.ndarray) -> np.ndarray` | Population-weighted voting: `(num_heads, seq_len, d_v)` to `(seq_len, d_v)`. Falls back to mean if voting disabled. |
| `reset` | `() -> None` | Reset all accumulators for a new sequence |
| `get_stats` | `() -> Dict[str, Any]` | Returns `step_count`, `hebbian`, `habituation`, `config` |

#### Example

```python
from corteX.engine.inference_hooks import InferenceHookPipeline, InferenceHookConfig
import numpy as np

pipeline = InferenceHookPipeline(InferenceHookConfig(
    enable_hebbian=True,
    enable_habituation=True,
    enable_temperature=True,
    enable_voting=True,
))

# During forward pass of each transformer layer
attention_logits = np.random.randn(8, 64, 64)  # (num_heads, seq, seq)
logits = pipeline.apply_pre_attention(attention_logits, layer_idx=0)

# After softmax
attention_weights = np.exp(logits) / np.sum(np.exp(logits), axis=-1, keepdims=True)
query = np.random.randn(8, 64, 128)
key = np.random.randn(8, 64, 128)
pipeline.apply_post_attention(attention_weights, query, key)

# Output aggregation
head_outputs = np.random.randn(8, 64, 128)
output = pipeline.apply_output_aggregation(head_outputs, attention_weights)

# Reset between sequences
pipeline.reset()
print(pipeline.get_stats())
```

---

## neuro_adapter

LoRA (Low-Rank Adaptation) framework for brain-inspired fine-tuning. Low-rank perturbations to specific projection matrices steer the model without full retraining -- analogous to synaptic plasticity at specific synapses.

### `LoRAConfig`

**Type**: `@dataclass`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `rank` | `int` | `8` | Rank of the low-rank matrices A and B |
| `alpha` | `float` | `16.0` | Scaling factor (scaling = alpha / rank) |
| `dropout` | `float` | `0.05` | Dropout probability during training |
| `target_modules` | `List[str]` | `["q_proj", "v_proj"]` | Model modules to apply LoRA to |
| `fan_in_fan_out` | `bool` | `False` | Whether weight matrix is stored transposed |
| `bias` | `str` | `"none"` | Bias mode: `"none"`, `"all"`, or `"lora_only"` |
| `modules_to_save` | `List[str]` | `[]` | Additional modules to save alongside LoRA weights |

Methods: `to_dict() -> Dict`, `from_dict(data) -> LoRAConfig`

### `NeuroAdapterConfig`

**Type**: `@dataclass`

Full training configuration for a neuroscience-inspired LoRA adapter.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_model_name` | `str` | `"meta-llama/Llama-3.1-8B"` | Base model to adapt |
| `lora_config` | `LoRAConfig` | `LoRAConfig()` | LoRA parameters |
| `neuroscience_objectives` | `List[str]` | `["synaptic_scaling", ...]` | Training objectives from neuroscience |
| `training_lr` | `float` | `2e-4` | Learning rate |
| `warmup_steps` | `int` | `100` | Warmup steps |
| `max_steps` | `int` | `1000` | Maximum training steps |
| `eval_steps` | `int` | `50` | Evaluation interval |
| `save_steps` | `int` | `100` | Checkpoint save interval |
| `batch_size` | `int` | `4` | Training batch size |
| `gradient_accumulation` | `int` | `4` | Gradient accumulation steps |
| `fp16` | `bool` | `True` | Use FP16 mixed precision |
| `output_dir` | `str` | `"./neuro_adapter_output"` | Output directory for checkpoints |

Methods: `to_dict() -> Dict`, `from_dict(data) -> NeuroAdapterConfig`

### `LoRAWeight`

A single LoRA adapter weight with matrices A (rank x in_features) and B (out_features x rank). Delta applied to input x: `x @ A^T @ B^T * scaling`.

#### Constructor

```python
LoRAWeight(module_name: str, lora_A: np.ndarray, lora_B: np.ndarray, scaling: float = 2.0)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `apply` | `(x: np.ndarray) -> np.ndarray` | Compute LoRA delta: `x @ A^T @ B^T * scaling` |
| `get_delta` | `() -> np.ndarray` | Full weight delta matrix: `B @ A * scaling` |
| `merge` | `(original_weight: np.ndarray) -> np.ndarray` | Merge delta into original: `original + B @ A * scaling` |
| `save` | `(path: str) -> None` | Save A, B matrices and metadata to a directory |
| `load` | `(path: str) -> LoRAWeight` | Class method: load from directory |

### `AdapterManager`

Manages multiple named LoRA adapters with load, save, merge, and inspection.

#### Constructor

```python
AdapterManager()
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `load_adapter` | `(path: str, name: str) -> None` | Load adapter from directory containing module subdirectories |
| `save_adapter` | `(path: str, name: str) -> None` | Save a named adapter to directory |
| `list_adapters` | `() -> List[str]` | List all loaded adapter names |
| `get_adapter` | `(name: str) -> Optional[Dict[str, LoRAWeight]]` | Get adapter by name |
| `get_adapter_info` | `(name: str) -> Dict[str, Any]` | Metadata and summary (num_modules, total_parameters, modules) |
| `merge_adapters` | `(adapter_names: List[str], weights: List[float], merged_name: str) -> Dict[str, LoRAWeight]` | Weighted combination of multiple adapters |

### `NeuroscienceAdapterSpec`

Defines neuroscience modifications trainable as LoRA adapters. Each method returns a spec with `target_modules`, `additional_params`, and `training_objective`.

| Method | Description |
|--------|-------------|
| `synaptic_scaling_spec()` | Per-head alpha scaling (homeostatic synaptic scaling) |
| `early_exit_spec()` | Early-exit classifiers for adaptive computation depth |
| `prediction_head_spec()` | Per-layer next-representation predictors (Friston's predictive coding) |
| `lateral_inhibition_spec()` | Cross-column lateral inhibition for head specialization |
| `list_all_specs()` | Return all available specs as a dict |

#### Example

```python
from corteX.engine.neuro_adapter import AdapterManager, LoRAWeight, NeuroscienceAdapterSpec
import numpy as np

# Create and apply a LoRA weight
lora = LoRAWeight("q_proj", lora_A=np.random.randn(8, 4096), lora_B=np.random.randn(4096, 8))
delta = lora.apply(input_tensor)
merged_weight = lora.merge(original_weight)

# Manage multiple adapters
manager = AdapterManager()
manager.load_adapter("./adapters/v1", name="v1")
manager.load_adapter("./adapters/v2", name="v2")
merged = manager.merge_adapters(["v1", "v2"], weights=[0.7, 0.3])
info = manager.get_adapter_info("merged")

# Explore neuroscience adapter specifications
specs = NeuroscienceAdapterSpec.list_all_specs()
for name, spec in specs.items():
    print(f"{name}: targets {spec.target_modules}, objective: {spec.training_objective}")
```

---

## training_collector

Training data collector that captures brain-state-driven LLM interactions for supervised fine-tuning (SFT), Direct Preference Optimization (DPO), and brain-conditioned training.

### `TrainingExample`

**Type**: `@dataclass`

A single training example from a brain-state-driven LLM interaction.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `timestamp` | `float` | `time.time()` | Unix timestamp of the interaction |
| `session_id` | `str` | `""` | Session identifier |
| `turn_number` | `int` | `0` | Turn number within the session |
| `brain_snapshot` | `Dict[str, Any]` | `{}` | Serialized brain state at this turn |
| `input_messages` | `List[Dict[str, Any]]` | `[]` | Input messages (role, content pairs) |
| `system_prompt` | `str` | `""` | System prompt used for this turn |
| `llm_parameters` | `Dict[str, float]` | `{}` | LLM parameters (temperature, top_p, etc.) |
| `llm_response` | `str` | `""` | The LLM's response text |
| `outcome` | `Dict[str, Any]` | `{}` | Outcome signals (quality_score, success, user_feedback) |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata |

Methods: `to_dict() -> Dict`, `from_dict(data) -> TrainingExample`

### `TrainingCollector`

Buffered collector that writes training examples to JSONL files. Auto-flushes at configurable intervals.

#### Constructor

```python
TrainingCollector(
    output_dir: str = "./training_data",
    max_buffer_size: int = 1000,
    auto_flush_interval: int = 100,
)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `collect` | `(example: TrainingExample) -> None` | Add example to buffer. Auto-flushes at interval or max size. |
| `flush` | `() -> str` | Write buffer to timestamped JSONL file. Returns path or empty string if empty. |
| `export_for_training` | `(format: str = "jsonl") -> str` | Export all data (buffer + flushed) to single file |
| `export_huggingface_format` | `() -> str` | Export as HuggingFace Dataset-compatible JSONL (instruction, input, output, brain_state) |
| `filter` | `(min_quality: Optional[float], outcome_type: Optional[str]) -> List[TrainingExample]` | Filter examples from flushed files and buffer |
| `get_stats` | `() -> Dict[str, Any]` | Returns `total_collected`, `total_flushed`, `buffer_size`, `flush_count`, `output_dir` |

### `BrainStateSerializer`

Serializes brain component state for embedding in TrainingExample objects. All methods are static.

| Method | Description |
|--------|-------------|
| `serialize_brain_snapshot(weight_engine, column_manager, prediction_engine, goal_tracker)` | Capture full brain state as a serializable dict |
| `serialize_weights(weight_engine)` | Capture weight state (uses `snapshot()` if available) |
| `serialize_column_state(column_manager)` | Capture column competition state (strips history) |
| `serialize_prediction_state(prediction_engine)` | Capture prediction accuracy and calibration |
| `deserialize_brain_snapshot(data)` | Reconstruct brain state dict from serialized data |

### `TrainingDataPipeline`

Formats collected examples for different training paradigms. All methods are static.

| Method | Signature | Description |
|--------|-----------|-------------|
| `create_sft_pairs` | `(examples) -> List[Dict[str, str]]` | Format as (instruction, response) pairs for SFT |
| `create_dpo_pairs` | `(examples, quality_threshold=0.7) -> List[Dict[str, str]]` | Format as (chosen, rejected) pairs for DPO. Groups by instruction, splits by quality threshold. |
| `create_brain_conditioned_pairs` | `(examples) -> List[Dict[str, str]]` | Format as (brain_state + instruction, response) pairs for brain-conditioned training |
| `compute_quality_labels` | `(examples) -> List[Tuple[TrainingExample, float]]` | Auto-label quality from outcome signals (weighted: 0.4 quality_score + 0.3 success + 0.2 feedback + 0.1 surprise) |

#### Example

```python
from corteX.engine.training_collector import (
    TrainingCollector, TrainingExample, TrainingDataPipeline
)

collector = TrainingCollector(output_dir="./data", max_buffer_size=500)

collector.collect(TrainingExample(
    session_id="s1",
    input_messages=[{"role": "user", "content": "Explain REST APIs"}],
    llm_response="REST APIs use HTTP methods...",
    outcome={"quality_score": 0.9, "success": True},
))

# Export for HuggingFace
path = collector.export_huggingface_format()

# Create training pairs
examples = collector.filter(min_quality=0.7)
sft_pairs = TrainingDataPipeline.create_sft_pairs(examples)
dpo_pairs = TrainingDataPipeline.create_dpo_pairs(examples, quality_threshold=0.7)
brain_pairs = TrainingDataPipeline.create_brain_conditioned_pairs(examples)
```

---

## early_exit

System 1/2 dual-process inference within a single local model. Kahneman's dual-process theory at the model architecture level: System 1 exits early from transformer layers (fast, heuristic), System 2 uses the full forward pass (slow, deliberate).

### `EarlyExitConfig`

**Type**: `@dataclass`

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `exit_layers` | `List[int]` | `[8, 16, 24]` | Transformer layers where exit classifiers are placed |
| `confidence_threshold` | `float` | `0.9` | Minimum calibrated confidence for early exit |
| `min_layer` | `int` | `4` | Earliest layer that can trigger an exit |
| `temperature` | `float` | `1.0` | Temperature for confidence estimation |
| `fallback_to_full` | `bool` | `True` | Whether to fall back to full pass if no exit |
| `system1_speedup_target` | `float` | `2.0` | Target speedup ratio for System 1 exits |
| `track_statistics` | `bool` | `True` | Whether to track exit statistics |

### `ExitClassifier`

Two-layer MLP classifier at a transformer exit point. Architecture: `hidden_dim -> hidden_dim//4 -> output_dim` with GELU activation.

#### Constructor

```python
ExitClassifier(hidden_dim: int, output_dim: int, layer_idx: int)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `classify` | `(hidden_state: np.ndarray) -> np.ndarray` | Forward pass returning raw logits. Input: `(hidden_dim,)`. |
| `estimate_confidence` | `(hidden_state: np.ndarray, temperature: float = 1.0) -> float` | Max softmax probability as confidence estimate |
| `save_weights` | `(path: str) -> None` | Persist classifier weights to `.npz` file |
| `load_weights` | `(path: str) -> None` | Load classifier weights from `.npz` file |

### `ConfidenceCalibrator`

Platt scaling for confidence calibration with online learning: `calibrated = sigmoid(a * raw + b)`. Maintains 10-bin ECE (Expected Calibration Error).

#### Constructor

```python
ConfidenceCalibrator(num_bins: int = 10, learning_rate: float = 0.05)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `calibrate` | `(raw_confidence: float) -> float` | Apply Platt scaling to raw confidence |
| `update` | `(raw_confidence: float, was_correct: bool) -> None` | Update parameters from a single outcome |
| `get_ece` | `() -> float` | Expected Calibration Error across bins |
| `get_bin_summary` | `() -> List[Dict[str, float]]` | Per-bin calibration diagnostics (range, avg_predicted, actual_accuracy, count) |
| `to_dict` / `from_dict` | -- | Serialization / deserialization |

### `DualProcessInference`

Orchestrates System 1/2 inference within a single local model. System 1 exits early when confidence exceeds threshold. System 2 uses all layers.

#### Constructor

```python
DualProcessInference(config: EarlyExitConfig)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_classifier` | `(layer_idx: int, classifier: ExitClassifier) -> None` | Attach an ExitClassifier at a specific layer |
| `should_exit` | `(hidden_state, layer_idx) -> Tuple[bool, float, str]` | Returns `(should_exit, confidence, "system1"\|"system2")` |
| `process_layer_output` | `(hidden_state, layer_idx) -> Optional[Tuple[np.ndarray, float]]` | Returns `(logits, confidence)` if exiting, `None` to continue |
| `record_full_pass` | `() -> None` | Record that inference used all layers (no early exit) |
| `get_system_decision` | `(task_complexity: float, context_signals: Dict) -> str` | Pre-route based on task signals (complexity, stakes, novelty, error_rate, time_pressure) |
| `record_outcome` | `(exit_layer: int, was_correct: bool) -> None` | Feed outcome back to calibrator |
| `get_stats` | `() -> Dict[str, Any]` | Returns `system1_ratio`, `avg_exit_layer`, `avg_confidence_at_exit`, `speedup_achieved`, `per_layer_exit_counts`, `calibration_ece`, `total_inferences` |

### `AdaptiveComputationController`

Dynamically adjusts confidence threshold for early exit based on task signals and outcome history.

#### Constructor

```python
AdaptiveComputationController(base_threshold: float = 0.9, adaptation_rate: float = 0.01)
```

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `compute_threshold` | `(task_signals: Dict[str, float]) -> float` | Dynamic threshold from complexity, stakes, time_pressure, user_patience. Range: `[0.5, 0.99]`. |
| `adapt` | `(exit_decision_was_correct: bool) -> None` | Update state after observing an exit outcome. Incorrect exits increase error penalty. |
| `get_stats` | `() -> Dict[str, Any]` | Returns `base_threshold`, `recent_correct_ema`, `error_penalty`, `total_adaptations` |
| `to_dict` / `from_dict` | -- | Serialization / deserialization |

### `create_dual_process_pipeline`

```python
def create_dual_process_pipeline(
    config: Optional[EarlyExitConfig] = None,
    hidden_dim: int = 768,
    output_dim: int = 32000,
) -> Tuple[DualProcessInference, AdaptiveComputationController]
```

Factory function that builds a fully configured early exit pipeline with classifiers at each exit layer, confidence calibration, and adaptive threshold control.

#### Example

```python
from corteX.engine.early_exit import create_dual_process_pipeline, EarlyExitConfig

engine, controller = create_dual_process_pipeline(
    config=EarlyExitConfig(exit_layers=[8, 16, 24], confidence_threshold=0.9),
    hidden_dim=4096,
    output_dim=128256,
)

# During inference: check at each exit layer
for layer_idx in range(32):
    hidden_state = transformer_layers[layer_idx](x)
    threshold = controller.compute_threshold({"complexity": 0.3, "stakes_level": 0.2})
    result = engine.process_layer_output(hidden_state, layer_idx)
    if result is not None:
        logits, confidence = result  # System 1 early exit
        break
else:
    engine.record_full_pass()

# Feedback for calibration
engine.record_outcome(exit_layer=8, was_correct=True)
controller.adapt(exit_decision_was_correct=True)

stats = engine.get_stats()
# {"system1_ratio": 0.65, "avg_exit_layer": 12.3, "speedup_achieved": 2.1}
```

---

## Performance Notes

- All inference hooks operate on NumPy arrays -- no PyTorch/TensorFlow dependency
- HebbianAccumulator matrix grows lazily up to `max_matrix_dim`
- Habituation tracking is per-layer (1000*head + layer_idx composite key)
- Temperature computation is stateless (no persistent state)
- Population voting normalization avoids division-by-zero with epsilon guards
- ExitClassifier uses GELU activation with the approximation formula (no erf dependency)
- ConfidenceCalibrator uses numerically stable sigmoid implementation

---

## See Also

- [Brain State Injector API](./brain-state-injector.md) -- How brain state reaches the LLM prompt
- [NeuroLlama Model API](../neurollama/neurollama-model.md) -- Full neuroscience-enhanced transformer
- [Calibration API](./calibration.md) -- System-level calibration (connects to ConfidenceCalibrator)
- [Game Theory API](./game-theory.md) -- DualProcessRouter for routing between models (complements early_exit)
