# NeuroLlama

Neuroscience-enhanced transformer architecture. Augments standard Llama-style attention with synaptic modulation, cortical columns, predictive coding, Hebbian learning, habituation, population coding, and early exit mechanisms.

## config

Central configuration dataclass parameterizing the standard Llama architecture and every neuroscience-inspired modification.

::: corteX.neurollama.config
    options:
      show_source: true
      members_order: source

---

## synaptic_attention

Three tiers of biologically-inspired attention modulation: per-head synaptic scaling, context-dependent neuromodulation, and full pairwise synaptic matrices.

::: corteX.neurollama.synaptic_attention
    options:
      show_source: true
      members_order: source

---

## cortical_columns

GQA key-value groups as functional cortical columns with lateral inhibition and hierarchical organization.

::: corteX.neurollama.cortical_columns
    options:
      show_source: true
      members_order: source

---

## predictive_coding

Higher layers predict lower-layer representations. Prediction errors drive learning and quantify surprise. Includes Contrastive Predictive Coding (CPC) for self-supervised learning.

::: corteX.neurollama.predictive_coding
    options:
      show_source: true
      members_order: source

---

## hebbian_attention

Within-sequence Hebbian learning for attention, reward-modulated three-factor plasticity, and fast-weight FFN layers.

::: corteX.neurollama.hebbian_attention
    options:
      show_source: true
      members_order: source

---

## habituation_layer

Stimulus-specific adaptation (SSA) for attention: per-head habituation, cross-layer cumulative decay, and novelty detection with dishabituation.

::: corteX.neurollama.habituation_layer
    options:
      show_source: true
      members_order: source

---

## population_output

Population-coded output aggregation: tuning-curve voting, confidence-weighted voting, entropy-based voting, and population vector decoding.

::: corteX.neurollama.population_output
    options:
      show_source: true
      members_order: source

---

## model

Full NeuroLlama model assembly: embedding, RMSNorm, RoPE, SwiGLU, NeuroLlamaBlock, NeuroLlamaModel, and factory presets.

::: corteX.neurollama.model
    options:
      show_source: true
      members_order: source

---

## training_objectives

Brain-inspired training loss functions: surprise, specialization, calibration, efficiency, coherence, composite multi-objective, and RLBF reward shaping.

::: corteX.neurollama.training_objectives
    options:
      show_source: true
      members_order: source

---

## Module Summary

### Configuration

| Class | Description |
|-------|------------|
| `NeuroLlamaConfig` | Complete configuration covering model architecture and all neuroscience toggles |
| `HabituationConfig` | Parameters governing habituation dynamics |
| `PopulationConfig` | Parameters for population-coded output aggregation |
| `HebbianConfig` | Configuration for Hebbian attention and fast-weight modules |
| `PredictiveCodingConfig` | Configuration for predictive coding layers |
| `TrainingObjectiveConfig` | Weights and hyper-parameters for all training objectives |

### Synaptic Attention

| Class | Description |
|-------|------------|
| `SynapticScaling` | Per-head learnable scaling factors (synaptic strength) |
| `NeuromodulatedAttention` | Context-dependent gating mimicking neuromodulators (dopamine, serotonin) |
| `SynapticModulationMatrix` | Full pairwise query-key modulation within a local attention window |
| `scaled_dot_product_attention()` | Standard scaled dot-product attention baseline (NumPy) |

### Cortical Columns

| Class | Description |
|-------|------------|
| `CorticalColumnAttention` | GQA groups as functional cortical columns with specialization divergence |
| `LateralInhibition` | Cross-column inhibitory connections preventing redundant representations |
| `HierarchicalColumnOrganizer` | Assigns roles to columns by layer depth (syntactic / semantic / abstract) |

### Predictive Coding

| Class | Description |
|-------|------------|
| `PredictionHead` | Per-layer linear projection that predicts the next layer's input |
| `PredictiveCodingLayer` | Transformer layer wrapper with top-down predictive coding |
| `ContrastivePredictiveCoding` | CPC / InfoNCE loss for self-supervised representation learning |
| `PredictionErrorSignal` | Manages prediction error signals across the full model |

### Hebbian Attention

| Class | Description |
|-------|------------|
| `HebbianAttention` | Attention with within-sequence Hebbian learning (co-activation matrix H) |
| `RewardModulatedHebbian` | Three-factor learning: pre x post x reward (STDP analogue) |
| `HebbianFastWeights` | Hebbian plasticity applied to FFN weights (rapid binding / short-term memory) |

### Habituation

| Class | Description |
|-------|------------|
| `HabituatingAttention` | Per-head habituation suppressing repeated attention patterns |
| `AttentionDecay` | Cross-layer cumulative decay preventing fixation |
| `NoveltyDetector` | Dishabituation boost for surprising tokens |

### Population Output

| Class | Description |
|-------|------------|
| `PopulationCodedAttention` | Gaussian tuning-curve population vectors |
| `ConfidenceWeightedVoting` | Learned per-head confidence estimators for weighted voting |
| `EntropyBasedVoting` | Attention entropy as confidence proxy (parameter-free) |
| `PopulationVectorDecoder` | Project population vector to vocabulary logits |

### Model Assembly

| Class | Description |
|-------|------------|
| `RMSNorm` | Root Mean Square layer normalization (Zhang & Sennrich, 2019) |
| `RotaryPositionEmbedding` | Rotary Position Embedding (RoPE) for Llama-style models |
| `SwiGLU` | SwiGLU feed-forward network (Shazeer, 2020) |
| `NeuroLlamaBlock` | Single transformer block with all neuroscience modifications |
| `NeuroLlamaModel` | Full model: embedding -> N blocks -> norm -> lm_head |
| `create_neurollama()` | Factory: create from preset name ("8B", "70B", "405B") or config |

### Training Objectives

| Class | Description |
|-------|------------|
| `SurpriseLoss` | Dopaminergic inter-layer prediction error |
| `SpecializationLoss` | Jensen-Shannon divergence between cortical columns |
| `CalibrationLoss` | Expected Calibration Error (ECE) |
| `EfficiencyLoss` | Metabolic cost (penalize layer usage) |
| `CoherenceLoss` | PFC goal-alignment (negative cosine similarity) |
| `NeuroCompositeLoss` | Multi-objective weighted combination with phase scheduling |
| `BrainInspiredReward` | RLBF reward shaping for reinforcement learning |
