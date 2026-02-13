# NeuroLlama Architecture

NeuroLlama is corteX's neuroscience-enhanced transformer architecture. It augments a standard Llama-3-style backbone with seven biological mechanisms that operate at the attention level, enabling on-prem models to exhibit adaptive, brain-like behavior without external API dependencies.

## What It Does

NeuroLlama replaces vanilla multi-head attention with a modular pipeline of neuroscience-inspired modifications. Each mechanism is independently toggleable, allowing operators to enable exactly the biological features their workload benefits from:

| Mechanism | Brain Analogy | Effect |
|-----------|--------------|--------|
| **Synaptic Modulation** | Synaptic strength (LTP/LTD) | Per-head scaling factors amplify or attenuate individual attention heads |
| **Cortical Columns** | Cortical minicolumns | GQA groups act as semi-independent processing units that specialize |
| **Predictive Coding** | Visual cortex V1/V2 | Higher layers predict lower-layer representations; mismatches drive learning |
| **Hebbian Attention** | Hebb's rule (STDP) | Co-activated attention patterns strengthen within a single sequence |
| **Habituation** | Stimulus-specific adaptation | Repeated attention patterns are suppressed; novel stimuli get boosted |
| **Population Coding** | Motor cortex population vectors | Head outputs are aggregated via confidence-weighted voting instead of concatenation |
| **Early Exit** | Dual process (System 1/2) | Confident layers can produce output early, skipping remaining computation |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Cortical Information Processing"
    The mammalian neocortex processes information through a hierarchical series of columns, each specialized for different features. Early visual areas (V1/V2) detect simple edges; later areas (V4/IT) recognize objects; prefrontal cortex handles abstract reasoning.

    Within each column, neurons communicate through synapses whose strength adapts via Hebbian learning ("cells that fire together wire together"). Habituation -- reduced response to repeated stimuli -- prevents neural saturation. And population coding in motor cortex uses the collective vote of thousands of neurons, each with a preferred direction, to accurately predict movement.

    NeuroLlama maps each of these mechanisms to transformer attention, creating a model that adapts within a single forward pass.

## Architecture Layers

NeuroLlama is organized into three integration layers, each adding deeper neuroscience modifications:

```
Layer 1: Standard Llama Backbone
  RMSNorm -> Multi-Head GQA -> SwiGLU FFN -> Residual

Layer 2: Inference-Time Hooks (no training required)
  HebbianAccumulator -> AttentionHabituation -> AdaptiveHeadTemperature -> PopulationWeightedVoting

Layer 3: Trained Neuroscience Modules (require fine-tuning)
  SynapticScaling -> CorticalColumnAttention -> PredictiveCodingLayer
  -> HabituatingAttention -> PopulationCodedAttention -> EarlyExit
```

### Layer 1: Llama Backbone

The base architecture follows Llama-3 conventions:

- **RMSNorm** (Zhang & Sennrich, 2019) for pre-norm layer normalization
- **Rotary Position Embedding (RoPE)** with theta=500,000 for long-context support
- **Grouped Query Attention (GQA)** with configurable KV head counts
- **SwiGLU** feed-forward networks (Shazeer, 2020)

```python
from corteX.neurollama import NeuroLlamaConfig
from corteX.neurollama.model import create_neurollama

# Create a model with Llama-3.1-8B dimensions
config = NeuroLlamaConfig(
    num_layers=32,
    hidden_dim=4096,
    num_heads=32,
    num_kv_heads=8,
    head_dim=128,
    ffn_dim=14336,
    vocab_size=128256,
)

model = create_neurollama(config)
```

### Layer 2: Inference-Time Hooks

These operate during the forward pass on any pre-trained model without additional training. They modify attention patterns in real-time:

```python
from corteX.engine.inference_hooks import InferenceHookPipeline, InferenceHookConfig

pipeline = InferenceHookPipeline(InferenceHookConfig(
    enable_hebbian=True,
    enable_habituation=True,
    enable_temperature=True,
    enable_voting=True,
))

# Pre-attention: habituation + adaptive temperature
logits = pipeline.apply_pre_attention(attention_logits, layer_idx=0)

# Post-attention: update Hebbian co-activation matrix
pipeline.apply_post_attention(attention_weights, query, key)

# Output: population-weighted aggregation
output = pipeline.apply_output_aggregation(head_outputs, attention_weights)
```

### Layer 3: Trained Modules

These require fine-tuning (via LoRA or full training) to reach optimal performance, but provide the deepest neuroscience integration.

## Synaptic Modulation

Three tiers of attention modulation, from simplest to most expressive:

### Static Synaptic Scaling

Each head gets a learnable scalar alpha that amplifies or attenuates its contribution. Analogous to synaptic strength -- some synapses transmit more strongly than others.

```python
from corteX.neurollama import SynapticScaling

scaling = SynapticScaling(num_heads=32)
modulated_scores = scaling.forward(attention_scores)  # (batch, heads, seq_q, seq_k)

# Strengthen or weaken a specific head
scaling.update_alpha(head_idx=5, delta=0.1)
```

**Reference**: Homeostatic synaptic scaling (Turrigiano, 2008) -- neurons globally adjust their synaptic strengths to maintain stable firing rates.

### Context-Dependent Neuromodulation

A two-layer network maps a context vector to per-head gating factors in [0, 1], mimicking how dopamine and serotonin provide global signals that modify local processing:

```python
from corteX.neurollama import NeuromodulatedAttention

neuromod = NeuromodulatedAttention(hidden_dim=4096, num_heads=32)
gates = neuromod.compute_modulation(context_vector)
# gates: (batch, 32) values in [0, 1]

gated_output = neuromod.forward(attention_output, context_vector)
```

**Reference**: Neuromodulatory systems (Doya, 2002) -- dopamine, serotonin, acetylcholine, and norepinephrine each modulate different aspects of learning and decision-making.

### Full Synaptic Matrix

The most expressive tier: a learned weight matrix computes pairwise synaptic strengths for each query-key pair within a local attention window.

```python
from corteX.neurollama import SynapticModulationMatrix

synapse = SynapticModulationMatrix(head_dim=128, window_size=512)
strengths = synapse.compute_synaptic_strength(query, key)
modulated = synapse.forward(attention_scores, query, key)
```

## Cortical Columns

GQA key-value groups are treated as functional **cortical columns** -- semi-independent processing units that specialize in different aspects of the input.

### Column Attention

Each GQA group processes its share of query heads independently. A Jensen-Shannon divergence metric measures how differently each column attends, driving specialization during training:

```python
from corteX.neurollama import CorticalColumnAttention

columns = CorticalColumnAttention(
    num_groups=8,         # 8 KV heads = 8 cortical columns
    heads_per_group=4,    # 32 query heads / 8 groups
    head_dim=128,
)

output, specialization_div = columns.forward(Q, K, V)
# specialization_div: higher = columns attend to different things (desirable)
```

### Lateral Inhibition

Cross-column inhibitory connections prevent redundant representations. Winner columns suppress losers through anti-Hebbian updates:

```python
from corteX.neurollama import LateralInhibition

inhibition = LateralInhibition(num_groups=8)
inhibited_outputs = inhibition.forward(group_outputs)

# Anti-Hebbian update: co-active columns develop mutual inhibition
inhibition.update_inhibition(group_activations, lr=0.01)
```

**Reference**: Lateral inhibition in cortex (Isaacson & Scanziani, 2011) -- inhibitory interneurons enforce competition between neighboring columns, promoting sparse, non-redundant representations.

### Hierarchical Organization

Layers are partitioned into three tiers mirroring the ventral visual hierarchy:

| Layer Range | Tier | Brain Analogy | Specialization |
|------------|------|---------------|----------------|
| 0-9 | Syntactic | V1/V2 | Low-level features, token patterns |
| 10-19 | Semantic | V4/IT | Mid-level meaning, phrase structure |
| 20+ | Abstract | PFC | High-level reasoning, goal alignment |

```python
from corteX.neurollama import HierarchicalColumnOrganizer

organizer = HierarchicalColumnOrganizer(num_layers=32, num_groups=8)
role = organizer.get_column_role(layer_idx=25, group_idx=3)
# "abstract"

targets = organizer.get_specialization_targets(layer_idx=15)
# {"tier": "semantic", "target_distribution": array([0.1, 0.7, 0.2]), ...}
```

## Predictive Coding

Higher layers generate predictions of lower-layer representations. When predictions do not match reality, "error neurons" compute the mismatch. This prediction error drives learning and quantifies surprise.

```python
from corteX.neurollama.predictive_coding import PredictiveCodingLayer, PredictiveCodingConfig

layer = PredictiveCodingLayer(
    hidden_dim=4096,
    config=PredictiveCodingConfig(
        prediction_weight=0.1,
        error_signal_decay=0.9,
    ),
)

output, prediction_for_below, surprise = layer.forward(
    x=layer_input,
    prediction_from_above=top_down_prediction,
)
```

The module also implements **Contrastive Predictive Coding (CPC)** for self-supervised representation learning:

```python
from corteX.neurollama.predictive_coding import ContrastivePredictiveCoding

cpc = ContrastivePredictiveCoding(
    hidden_dim=4096,
    num_negatives=16,
    prediction_horizon=1,
)

loss = cpc.compute_loss(hidden_states, target_embeddings)
```

**References**: Rao & Ballard (1999) -- predictive coding in visual cortex; van den Oord et al. (2018) -- CPC / InfoNCE loss; arXiv:2503.04416 -- Transformer World Models with CPC (2025).

## Hebbian Attention

Within-sequence Hebbian learning creates "short-term memory" that enhances in-context learning. A co-activation matrix H biases future attention toward previously attended patterns:

$$A = \text{softmax}\left(\frac{QK^T}{\sqrt{d}} + \eta \cdot Q H K^T\right)$$

```python
from corteX.neurollama.hebbian_attention import HebbianAttention, HebbianConfig

hebbian = HebbianAttention(
    head_dim=128,
    config=HebbianConfig(
        learning_rate=0.01,
        decay_rate=0.99,
        max_hebbian_norm=10.0,
    ),
)

output = hebbian.forward(Q, K, V)
strength = hebbian.get_hebbian_strength()  # Frobenius norm of H
```

### Reward-Modulated Hebbian Learning

Three-factor learning (pre x post x reward) gates plasticity so only rewarded co-activations are strengthened -- the neural analogue of spike-timing-dependent plasticity (STDP):

```python
from corteX.neurollama.hebbian_attention import RewardModulatedHebbian

reward_hebb = RewardModulatedHebbian(head_dim=128)
output = reward_hebb.forward(Q, K, V, reward=0.8)
```

**References**: Hebb (1949) -- *The Organization of Behaviour*; Eliasmith et al. (2024) -- Short-term Hebbian Learning as Transformer Attention, PLOS Computational Biology; arXiv:2510.21908 -- In-Context Memory with Hebbian Plasticity (2025).

## Habituation

Stimulus-specific adaptation (SSA) suppresses attention to repeated patterns while staying responsive to novel stimuli. Three components work together:

1. **HabituatingAttention** -- per-head pattern counting and suppression
2. **AttentionDecay** -- cross-layer cumulative decay preventing fixation
3. **NoveltyDetector** -- dishabituation boost for surprising tokens

```python
from corteX.neurollama import HabituatingAttention, HabituationConfig

hab = HabituatingAttention(
    num_heads=32,
    max_seq_len=4096,
    config=HabituationConfig(
        habituation_rate=0.1,
        recovery_rate=0.01,
        attention_threshold=0.1,
        max_habituation=0.9,
    ),
)

habituated_scores = hab.forward(attention_scores, seq_len=512)
habituation_map = hab.get_habituation_map()
```

**Reference**: Ulanovsky et al. (2003) -- stimulus-specific adaptation in the auditory cortex of the awake rat.

## Population-Coded Output

Instead of naive head concatenation, attention head outputs are aggregated via biologically-inspired voting mechanisms:

### Confidence-Weighted Voting

Learned per-head confidence estimators weight each head's contribution. Heads below a confidence threshold are silenced entirely (lateral inhibition):

```python
from corteX.neurollama import ConfidenceWeightedVoting

voting = ConfidenceWeightedVoting(num_heads=32, head_dim=128)
aggregated = voting.forward(head_outputs)  # (batch, head_dim)
```

### Tuning-Curve Population Coding

Each head has a preferred direction in feature space. Activation follows a Gaussian tuning curve, and the population vector is the activation-weighted average -- exactly how motor cortex represents movement:

```python
from corteX.neurollama import PopulationCodedAttention, PopulationConfig

pop = PopulationCodedAttention(PopulationConfig(
    num_heads=32,
    head_dim=128,
    hidden_dim=4096,
    voting_method="tuning_curve",
))

output = pop.forward(head_outputs, head_contexts)  # (batch, hidden_dim)
```

### Entropy-Based Voting

Parameter-free approach using attention entropy as a confidence proxy. Low entropy (confident heads) get higher voting weight:

```python
from corteX.neurollama import EntropyBasedVoting

voting = EntropyBasedVoting(num_heads=32, temperature=1.0)
aggregated = voting.forward(head_outputs, attention_weights)
```

**Reference**: Georgopoulos et al. (1986) -- neuronal population coding of movement direction, Science.

## Early Exit (System 1/2)

Kahneman's dual-process theory at the model architecture level. Confident layers can produce output early (System 1), while uncertain inputs traverse the full layer stack (System 2):

```python
from corteX.engine.early_exit import create_dual_process_pipeline, EarlyExitConfig

engine, controller = create_dual_process_pipeline(
    config=EarlyExitConfig(
        exit_layers=[8, 16, 24],
        confidence_threshold=0.9,
        min_layer=4,
    ),
    hidden_dim=4096,
    output_dim=128256,
)

# During inference, check at each exit layer
result = engine.process_layer_output(hidden_state, layer_idx=8)
if result is not None:
    logits, confidence = result
    # System 1 early exit with confidence
```

The `AdaptiveComputationController` dynamically adjusts the exit threshold based on task complexity, error rates, and time pressure.

**Reference**: Kahneman (2011) -- *Thinking, Fast and Slow*; Scardapane et al. (2020) -- early exit transformers for efficient inference.

## NeuroLlama Model Assembly

The full model combines all mechanisms into a single forward pass:

```python
from corteX.neurollama.model import create_neurollama, NeuroLlamaModel
from corteX.neurollama import NeuroLlamaConfig
import numpy as np

config = NeuroLlamaConfig(
    num_layers=32,
    hidden_dim=4096,
    enable_synaptic_modulation=True,
    enable_cortical_columns=True,
    enable_predictive_coding=True,
    enable_hebbian_attention=True,
    enable_habituation=True,
    enable_population_coding=True,
    enable_early_exit=True,
    exit_layers=[8, 16, 24],
)

model = create_neurollama(config)

# Forward pass
result = model.forward(input_ids=np.array([1, 2, 3, 4]))
logits = result["logits"]           # (1, seq_len, vocab_size)
exit_layer = result["exit_layer"]   # which layer exited (or num_layers if full pass)
system_type = result["system_type"] # "system1" or "system2"
surprises = result["surprises"]     # per-layer prediction error
```

### Model Presets

Three standard configurations matching Llama-3.1 model sizes:

| Preset | Layers | Hidden Dim | Heads | KV Heads | FFN Dim |
|--------|--------|-----------|-------|----------|---------|
| `"8B"` | 32 | 4096 | 32 | 8 | 14336 |
| `"70B"` | 80 | 8192 | 64 | 8 | 28672 |
| `"405B"` | 126 | 16384 | 128 | 16 | 53248 |

```python
model_8b = create_neurollama("8B")
model_70b = create_neurollama("70B")
```

## Training Objectives

NeuroLlama defines five brain-inspired training losses:

| Loss | Brain Analogy | Objective |
|------|--------------|-----------|
| **SurpriseLoss** | Dopaminergic prediction error | Minimize inter-layer prediction error |
| **SpecializationLoss** | Column competition | Maximize JS divergence between cortical columns |
| **CalibrationLoss** | Metacognition (ECE) | Align confidence with actual accuracy |
| **EfficiencyLoss** | Metabolic cost | Penalize using more layers than necessary |
| **CoherenceLoss** | PFC goal maintenance | Maximize cosine similarity between output and goal embedding |

```python
from corteX.neurollama.training_objectives import NeuroCompositeLoss, TrainingObjectiveConfig

loss_fn = NeuroCompositeLoss(TrainingObjectiveConfig(
    prediction_error_weight=0.1,
    specialization_weight=0.01,
    calibration_weight=0.01,
    efficiency_weight=0.05,
    coherence_weight=0.05,
))

total, components = loss_fn.compute(
    predictions=predictions,
    targets=targets,
    attention_distributions=group_attentions,
    confidences=confidences,
    correctness=correctness,
    layer_usage=layer_mask,
    hidden_states=final_hidden,
    goal_embedding=goal_vec,
)

# Adjust weights by training phase
loss_fn.adjust_weights(phase=2)  # 1=pretrain, 2=continued, 3=finetune
```

## Configuration

All neuroscience features are controlled through `NeuroLlamaConfig`:

```python
from corteX.neurollama import NeuroLlamaConfig

config = NeuroLlamaConfig(
    # Architecture
    num_layers=32,
    hidden_dim=4096,
    num_heads=32,
    num_kv_heads=8,

    # Toggle each neuroscience mechanism
    enable_synaptic_modulation=True,
    synaptic_mode="context_dependent",  # "static", "context_dependent", "full_matrix"
    enable_cortical_columns=True,
    enable_lateral_inhibition=True,
    enable_predictive_coding=True,
    enable_hebbian_attention=True,
    enable_habituation=True,
    enable_population_coding=True,
    enable_early_exit=True,
    exit_layers=[8, 16, 24],
    exit_confidence_threshold=0.9,
)

# Serialize / deserialize
json_str = config.to_json()
config2 = NeuroLlamaConfig.from_json(json_str)

# Override specific fields
fast_config = config.with_overrides(enable_early_exit=False, hebbian_lr=0.001)
```

## When to Use NeuroLlama

| Scenario | Recommended Configuration |
|----------|--------------------------|
| General-purpose on-prem agent | Layer 2 hooks only (no training required) |
| Domain-specialized agent | Layer 3 with LoRA fine-tuning |
| Low-latency inference | Enable early exit with System 1/2 routing |
| Long-context processing | Enable habituation to prevent attention fixation |
| Multi-domain expert | Enable cortical columns with lateral inhibition |
| Exploration tasks | Enable Hebbian attention for in-context memory |
