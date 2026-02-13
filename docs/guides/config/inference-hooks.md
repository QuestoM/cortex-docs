# How to Configure Inference-Time Brain Hooks

This guide shows you how to configure corteX's inference-time brain hooks -- neuroscience-inspired modifications that alter attention patterns during the forward pass of local models, without any additional training or fine-tuning.

## Overview

Layer 2 inference hooks sit between your local model's attention layers and the final output. They implement four biological mechanisms:

1. **HebbianAccumulator** -- strengthens co-activated attention patterns within a sequence
2. **AttentionHabituation** -- suppresses repeated attention patterns, freeing capacity for novel tokens
3. **AdaptiveHeadTemperature** -- per-head temperature scaling based on attention entropy
4. **PopulationWeightedVoting** -- confidence-weighted head aggregation instead of naive concatenation

All four hooks are orchestrated by the `InferenceHookPipeline`.

## Quick Start

```python
from corteX.engine.inference_hooks import InferenceHookPipeline, InferenceHookConfig
import numpy as np

# Create pipeline with all hooks enabled
pipeline = InferenceHookPipeline(InferenceHookConfig())

# Simulate attention logits: (num_heads, seq_len, seq_len)
num_heads, seq_len = 32, 128
attention_logits = np.random.randn(num_heads, seq_len, seq_len).astype(np.float32)

# 1. Pre-attention: habituation + temperature adjustment
modified_logits = pipeline.apply_pre_attention(attention_logits, layer_idx=0)

# 2. Compute softmax to get attention weights
weights = np.exp(modified_logits) / np.sum(np.exp(modified_logits), axis=-1, keepdims=True)

# 3. Post-attention: update Hebbian co-activation matrix
query = np.random.randn(num_heads, seq_len, 128)
key = np.random.randn(num_heads, seq_len, 128)
pipeline.apply_post_attention(weights, query, key)

# 4. Output aggregation: population-weighted voting
head_outputs = np.random.randn(num_heads, seq_len, 128)
aggregated = pipeline.apply_output_aggregation(head_outputs, weights)
# aggregated: (seq_len, 128) -- single fused representation

# Reset between sequences
pipeline.reset()
```

## Configuring Individual Hooks

### Hebbian Accumulator

The Hebbian accumulator tracks co-activation patterns across attention steps. Positions that are frequently attended together receive a boost in future attention scores.

```python
from corteX.engine.inference_hooks import HebbianConfig, HebbianAccumulator

config = HebbianConfig(
    decay=0.99,              # Exponential decay (prevents runaway potentiation)
    modulation_strength=0.1, # How much H biases attention scores
    max_matrix_dim=2048,     # Maximum co-activation matrix size
)

hebbian = HebbianAccumulator(config)

# Update from observed attention
hebbian.update(query, key, attention_weights)

# Modulate future attention scores
modulated = hebbian.modulate(new_attention_scores)

# Check learning progress
stats = hebbian.get_stats()
# {"update_count": 5, "matrix_allocated": True, "h_frobenius_norm": 2.34}
```

**Tuning guidance**:

- Increase `modulation_strength` (e.g., 0.2-0.5) for tasks where in-context learning is critical (few-shot prompting, code completion)
- Decrease `modulation_strength` (e.g., 0.01-0.05) for tasks requiring diverse attention (creative writing, brainstorming)
- Lower `decay` (e.g., 0.9) to forget patterns faster; raise it (e.g., 0.999) to maintain longer-term memory within a sequence

### Attention Habituation

Suppresses attention to heavily-attended tokens, freeing capacity for novel stimuli. Implements stimulus-specific adaptation (SSA) from auditory neuroscience.

```python
from corteX.engine.inference_hooks import HabituationConfig, AttentionHabituation

config = HabituationConfig(
    habituation_rate=0.1,  # Speed of suppression
    recovery_rate=0.01,    # Speed of dishabituation for novel tokens
)

habituation = AttentionHabituation(config)

# Apply to pre-softmax attention scores at a specific layer
habituated = habituation.apply_habituation(
    attention_scores,  # (seq_len, seq_len)
    layer_idx=0,
)
```

**Tuning guidance**:

- Increase `habituation_rate` (e.g., 0.3) for long documents where attention fixation is a problem
- Decrease `habituation_rate` (e.g., 0.01) for short prompts where all tokens matter
- Increase `recovery_rate` (e.g., 0.1) for tasks with frequent topic changes

### Adaptive Head Temperature

Scales each head's attention logits by a temperature derived from its entropy. Confident heads (low entropy) get sharpened; uncertain heads (high entropy) get softened:

```python
from corteX.engine.inference_hooks import TemperatureConfig, AdaptiveHeadTemperature

config = TemperatureConfig(
    alpha=0.5,     # Sensitivity to entropy differences
    temp_min=0.5,  # Minimum temperature (sharpest)
    temp_max=2.0,  # Maximum temperature (softest)
)

temperature = AdaptiveHeadTemperature(config)

# Compute per-head temperatures
temps = temperature.compute_temperatures(attention_logits)
# temps: (num_heads,)

# Apply temperatures to logits
scaled = temperature.apply_temperatures(attention_logits)
```

**Tuning guidance**:

- Higher `alpha` (e.g., 1.0) creates more extreme temperature differences between heads
- Narrow the `temp_min`/`temp_max` range (e.g., 0.8-1.2) for more conservative behavior
- Widen the range (e.g., 0.3-3.0) to let the system be more aggressive

### Population-Weighted Voting

Aggregates head outputs by confidence instead of simple concatenation. Uncertain heads are down-weighted; optionally, heads below a threshold are suppressed entirely:

```python
from corteX.engine.inference_hooks import VotingConfig, PopulationWeightedVoting

config = VotingConfig(
    suppress_threshold=0.15,  # Suppress heads below this confidence
    suppress_below=True,      # Enable lateral inhibition
)

voting = PopulationWeightedVoting(config)

# Compute per-head confidence
confidences = voting.compute_confidence(head_outputs, attention_weights)

# Aggregate
aggregated = voting.aggregate(head_outputs, confidences)
# aggregated: (seq_len, d_v)
```

**Tuning guidance**:

- Increase `suppress_threshold` (e.g., 0.3) to silence more uncertain heads (more aggressive pruning)
- Set `suppress_below=False` to keep all heads contributing (softer aggregation)

## Selective Hook Enabling

You can enable or disable individual hooks without changing the pipeline structure:

```python
config = InferenceHookConfig(
    enable_hebbian=True,       # Keep co-activation tracking
    enable_habituation=True,   # Keep novelty bias
    enable_temperature=False,  # Disable adaptive temperature
    enable_voting=False,       # Use simple averaging instead
)

pipeline = InferenceHookPipeline(config)
```

## Integrating with Local Model Serving

### With Ollama / vLLM

When using local models through an OpenAI-compatible API, inference hooks operate at the corteX orchestrator level rather than inside the model's forward pass:

```python
import cortex

engine = cortex.Engine(
    providers={
        "local": {
            "base_url": "http://localhost:11434/v1",
            "api_key": "ollama",
        }
    },
    inference_hooks=InferenceHookConfig(
        enable_hebbian=True,
        enable_habituation=True,
    ),
)
```

### With Direct Model Access

When you have direct access to model internals (e.g., transformers library), hooks integrate at the attention layer level:

```python
# Pseudo-code for PyTorch integration
class NeuroAttention(nn.Module):
    def __init__(self, base_attention, hook_config):
        super().__init__()
        self.base = base_attention
        self.hooks = InferenceHookPipeline(hook_config)

    def forward(self, hidden_states, attention_mask=None):
        # Get Q, K, V from base attention
        Q, K, V = self.base.project(hidden_states)

        # Pre-attention hooks
        logits = Q @ K.transpose(-2, -1) / math.sqrt(d_k)
        logits_np = logits.detach().cpu().numpy()
        logits_np = self.hooks.apply_pre_attention(logits_np, layer_idx=self.layer_idx)

        # Continue with modified logits...
```

## Monitoring Hook Activity

All hooks expose statistics for observability:

```python
stats = pipeline.get_stats()
# {
#     "step_count": 42,
#     "hebbian": {"update_count": 42, "matrix_allocated": True, "h_frobenius_norm": 3.14},
#     "habituation": {"tracked_layers": 32, "layer_indices": [0, 1, 2, ...]},
#     "config": {
#         "enable_hebbian": True,
#         "enable_habituation": True,
#         "enable_temperature": True,
#         "enable_voting": True,
#     }
# }
```

## LoRA Adapter Integration

For deeper integration, Layer 2 hooks work alongside LoRA adapters that encode neuroscience objectives:

```python
from corteX.engine.neuro_adapter import (
    AdapterManager,
    NeuroscienceAdapterSpec,
    NeuroAdapterConfig,
    LoRAConfig,
)

# Define a neuroscience-motivated adapter
spec = NeuroscienceAdapterSpec.synaptic_scaling_spec()
# target_modules: ["q_proj", "k_proj", "v_proj", "o_proj"]
# training_objective: "synaptic_scaling_loss"

# Configure LoRA training
adapter_config = NeuroAdapterConfig(
    base_model_name="meta-llama/Llama-3.1-8B",
    lora_config=LoRAConfig(rank=8, alpha=16.0, dropout=0.05),
    neuroscience_objectives=["synaptic_scaling", "prediction_error"],
)

# Manage trained adapters
manager = AdapterManager()
manager.load_adapter("./adapters/neuro_v1", name="neuro_v1")
info = manager.get_adapter_info("neuro_v1")
```

## Training Data Collection

The `TrainingCollector` captures brain-state-driven interactions for fine-tuning:

```python
from corteX.engine.training_collector import (
    TrainingCollector,
    TrainingExample,
    TrainingDataPipeline,
)

collector = TrainingCollector(output_dir="./training_data")

# Collect examples during live inference
example = TrainingExample(
    session_id="sess_123",
    brain_snapshot={"weights": {...}, "prediction": {...}},
    input_messages=[{"role": "user", "content": "Explain quantum computing"}],
    llm_response="Quantum computing uses qubits...",
    outcome={"quality_score": 0.85, "success": True},
)
collector.collect(example)

# Export for training
path = collector.export_huggingface_format()

# Create DPO training pairs
pipeline = TrainingDataPipeline()
dpo_pairs = pipeline.create_dpo_pairs(examples, quality_threshold=0.7)
```

## Early Exit Configuration

Configure System 1/2 dual-process inference at the model level:

```python
from corteX.engine.early_exit import (
    create_dual_process_pipeline,
    EarlyExitConfig,
    AdaptiveComputationController,
)

engine, controller = create_dual_process_pipeline(
    config=EarlyExitConfig(
        exit_layers=[8, 16, 24],       # Layers with exit classifiers
        confidence_threshold=0.9,       # Min confidence for early exit
        min_layer=4,                    # Never exit before layer 4
        system1_speedup_target=2.0,     # Target speedup over full pass
    ),
    hidden_dim=4096,
    output_dim=128256,
)

# Dynamic threshold adjustment based on task signals
threshold = controller.compute_threshold({
    "complexity": 0.8,      # High complexity -> higher threshold
    "stakes_level": 0.9,    # High stakes -> higher threshold
    "time_pressure": 0.3,   # Low time pressure -> higher threshold
})

# After observing outcome, adapt the controller
controller.adapt(exit_decision_was_correct=True)

# Monitor performance
stats = engine.get_stats()
# {"system1_ratio": 0.65, "avg_exit_layer": 12.3, "speedup_achieved": 2.1, ...}
```
