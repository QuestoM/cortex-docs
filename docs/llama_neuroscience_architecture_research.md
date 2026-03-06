# Neuroscience-Inspired Modifications to Meta's Llama Architecture
## Deep Technical Research Report

**Date**: February 2026
**Scope**: Llama 3.1/4 architecture analysis + neuroscience-inspired transformer modifications
**For**: corteX SDK - Exploring whether brain-inspired patterns can be embedded at the model level

---

## Table of Contents

1. [Llama Architecture Deep Dive](#1-llama-architecture-deep-dive)
2. [Neuroscience-Inspired Architecture Modifications](#2-neuroscience-inspired-architecture-modifications)
3. [Implementation Feasibility Matrix](#3-implementation-feasibility-matrix)
4. [Research Papers and Prior Art](#4-research-papers-and-prior-art)
5. [Custom Training Objectives](#5-custom-training-objectives)
6. [Synthesis: A Neuroscience-Enhanced Llama Architecture](#6-synthesis)

---

## 1. Llama Architecture Deep Dive

### 1.1 Llama 3.1 Family Architecture (Dense Transformers)

Llama 3.1 represents Meta's culmination of the dense (non-MoE) transformer approach. It is an autoregressive, decoder-only transformer with the following key modifications from the vanilla Transformer:

- **RMSNorm** instead of LayerNorm (pre-normalization)
- **SwiGLU** activation instead of ReLU/GELU in the FFN
- **Rotary Position Embeddings (RoPE)** instead of absolute/learned position embeddings
- **Grouped Query Attention (GQA)** instead of standard Multi-Head Attention (MHA)
- **No bias terms** in linear layers

#### Concrete Architecture Parameters

```
                    Llama 3.1 8B    Llama 3.1 70B    Llama 3.1 405B
--------------------------------------------------------------------
Layers (L)              32              80               126
Hidden dim (d)         4096            8192             16384
Attention heads (h)     32              64               128
KV heads (h_kv)          8               8                16
Head dim (d_h)         128             128               128
FFN dim (d_ff)        14336           28672             53248
Vocab size           128256          128256            128256
Context length        128K            128K              128K
Parameters            8.03B           70.6B             405B
GQA ratio (h/h_kv)     4:1             8:1               8:1
```

#### Single Transformer Block Diagram

```
                   Input Tokens
                       |
                       v
             +-------------------+
             |   RMSNorm (pre)   |
             +-------------------+
                       |
                       v
             +-------------------+
             |  Grouped Query    |
             |  Attention (GQA)  |  <-- RoPE applied to Q, K
             |  h_q heads, h_kv  |
             |  KV heads         |
             +-------------------+
                       |
                   + residual
                       |
                       v
             +-------------------+
             |   RMSNorm (pre)   |
             +-------------------+
                       |
                       v
             +-------------------+
             |    SwiGLU FFN     |
             |  W_gate, W_up     |
             |  -> Swish gate    |
             |  -> W_down        |
             +-------------------+
                       |
                   + residual
                       |
                       v
                   Next Layer
```

### 1.2 RoPE (Rotary Position Embeddings)

**Paper**: "RoFormer: Enhanced Transformer with Rotary Position Embedding" (Su et al., 2021)

RoPE encodes absolute position through rotation matrices while naturally capturing relative position in the attention dot product.

#### Mathematical Formulation

For a d-dimensional embedding, RoPE organizes dimensions into d/2 pairs. For each pair i at position m:

```
R(m, theta_i) = | cos(m * theta_i)   -sin(m * theta_i) |
                | sin(m * theta_i)    cos(m * theta_i) |

where theta_i = 10000^(-2i/d)     (base frequency)
```

Applied to query q and key k at positions m and n respectively:

```
q_rot(m) = R(m) * q
k_rot(n) = R(n) * k

Attention(m, n) = q_rot(m)^T * k_rot(n)
               = q^T * R(m)^T * R(n) * k
               = q^T * R(n - m) * k          <-- depends only on relative position!
```

**Key Properties:**
- Zero additional parameters (purely mathematical transformation)
- Naturally encodes relative position in the dot product
- Decays attention with distance (long-range tokens get less attention naturally)
- Frequency spectrum: low-frequency pairs capture long-range patterns, high-frequency pairs capture local patterns
- Can be extended to longer contexts via frequency scaling (NTK-aware RoPE, YaRN)

**Llama 3.1 Specific**: Uses rope_theta=500000 (increased from 10000 in Llama 2) to support 128K context. This stretches the frequency spectrum, allowing better long-range attention.

### 1.3 Grouped Query Attention (GQA)

**Paper**: "GQA: Training Generalized Multi-Query Attention from Multi-Head Checkpoints" (Ainslie et al., 2023)

GQA is an interpolation between Multi-Head Attention (MHA) and Multi-Query Attention (MQA):

```
MHA:   h query heads, h key heads, h value heads     (full)
GQA:   h query heads, g key heads, g value heads      (grouped, g < h)
MQA:   h query heads, 1 key head,  1 value head       (minimal)
```

#### How GQA Works

```
Query heads:  Q1  Q2  Q3  Q4 | Q5  Q6  Q7  Q8 | ...  (h = 32 heads)
              \___\___|___/     \___\___|___/
                  |                   |
KV heads:        KV1                 KV2         ...   (h_kv = 8 heads)
              (shared by 4          (shared by 4
               query heads)          query heads)

GQA ratio = h / h_kv = 32 / 8 = 4 query heads per KV group
```

**KV Cache Impact**: For Llama 3.1 405B with 128 query heads and 16 KV heads:
- MHA would require: 128 * 128 (head_dim) * 2 (K+V) * 126 (layers) = ~4.1 GB per 1K tokens
- GQA requires:      16 * 128 * 2 * 126 = ~516 MB per 1K tokens (8x reduction)

Each KV group functions somewhat independently -- different groups can learn different attention patterns, which is directly relevant to the "cortical columns" analogy explored in Section 2.

### 1.4 SwiGLU Activation

**Paper**: "GLU Variants Improve Transformer" (Shazeer, 2020)

Standard transformer FFN:
```
FFN(x) = W_2 * ReLU(W_1 * x)
```

SwiGLU FFN:
```
FFN(x) = W_down * [Swish(x * W_gate) (element-wise multiply) (x * W_up)]

where Swish(x) = x * sigmoid(beta * x)   (beta=1 in practice, a.k.a. SiLU)
```

This introduces a **gating mechanism**: the W_gate projection determines which neurons fire (via Swish), while W_up produces the values. The element-wise product means the gate controls information flow -- analogous to **synaptic gating** in neuroscience.

**Parameter count**: SwiGLU requires 3 weight matrices (W_gate, W_up, W_down) vs. 2 for standard FFN. To maintain similar parameter counts, the FFN intermediate dimension is reduced by a factor of 2/3:
```
Standard FFN: d_ff = 4 * d_model
SwiGLU:       d_ff = (8/3) * d_model   (rounded to nearest multiple of 256)

Llama 3.1 8B:   d=4096,  d_ff=14336  (ratio = 3.5x, close to 8/3 * 4096 = 10923 but larger)
Llama 3.1 405B:  d=16384, d_ff=53248  (ratio = 3.25x)
```

### 1.5 RMSNorm

**Paper**: "Root Mean Square Layer Normalization" (Zhang & Sennrich, 2019)

Standard LayerNorm:
```
LayerNorm(x) = gamma * (x - mean(x)) / sqrt(var(x) + eps) + beta
```

RMSNorm:
```
RMSNorm(x) = gamma * x / sqrt(mean(x^2) + eps)

where mean(x^2) = (1/d) * sum(x_i^2)   for i=1..d
```

**Key difference**: RMSNorm removes the mean-centering (re-centering) step, keeping only the re-scaling. This:
- Saves ~7-64% compute depending on model size
- Provides re-scaling invariance
- Implicitly adapts learning rate
- Works as well or better than LayerNorm empirically

Llama uses **pre-normalization** (RMSNorm applied BEFORE attention and FFN, not after), which improves training stability.

### 1.6 KV Cache Mechanics

During autoregressive generation, each new token only needs to compute attention against all previous tokens. Without caching, this means recomputing K and V projections for all previous tokens at every step.

```
Step 1: Process token t1 -> compute K1, V1 -> store in cache
Step 2: Process token t2 -> compute K2, V2 -> attend to [K1,K2], [V1,V2]
Step 3: Process token t3 -> compute K3, V3 -> attend to [K1,K2,K3], [V1,V2,V3]
...
Step n: Compute Kn, Vn -> attend to [K1..Kn], [V1..Vn]

KV Cache memory per token per layer = 2 * h_kv * d_h * sizeof(dtype)
Total KV cache = seq_len * num_layers * 2 * h_kv * d_h * sizeof(dtype)
```

For Llama 3.1 405B (FP16):
```
Per token per layer: 2 * 16 * 128 * 2 bytes = 8 KB
Per token total:     8 KB * 126 layers = ~1 MB
For 128K context:    128000 * 1 MB = ~128 GB  (just for KV cache!)
```

This is why GQA (reducing h_kv) and quantized KV caches (FP8, INT4) are critical.

### 1.7 Llama 4 Architecture (MoE + iRoPE)

Llama 4 represents a major architectural shift from dense to Mixture-of-Experts (MoE):

#### Llama 4 Model Configurations

```
                    Llama 4 Scout    Llama 4 Maverick    Llama 4 Behemoth
-------------------------------------------------------------------------
Active params          17B               17B               ~288B (est.)
Total params          109B              400B              ~2T (est.)
Layers                 48                48                 ~?
Hidden dim           5120              5120                ~?
Attention heads        40                40                 ~?
KV heads                8                 8                 ~?
Head dim              128               128                ~?
FFN dim (MoE)        8192              8192                ~?
FFN dim (dense)     16384             16384                ~?
Num experts            16               128                ~?
Experts per token       1                 1                 ~?
Context length       256K            1M (claimed)          ~?
Architecture        Full MoE      Interleaved MoE       Teacher
```

#### iRoPE (Interleaved Rotary Position Embedding)

Llama 4's key innovation -- interleaving layers WITH and WITHOUT positional encoding:

```
Layer 1:  RoPE + Chunked Attention   (local attention with position info)
Layer 2:  RoPE + Chunked Attention
Layer 3:  RoPE + Chunked Attention
Layer 4:  NoPE + Full Causal Mask    (no position encoding, global attention)
Layer 5:  RoPE + Chunked Attention
...
Pattern: Every 4th layer is a NoPE (No Positional Encoding) layer
```

**Why this matters for neuroscience analogy:**
- RoPE layers act like **local processing** (attending to nearby tokens with position awareness) -- analogous to receptive fields in visual cortex
- NoPE layers act like **global integration** (attending to entire context without position bias) -- analogous to associative cortex binding distant representations
- This is remarkably similar to the brain's alternation between local cortical processing and long-range white-matter connections

#### MoE Router Architecture

```
Input token x
      |
      v
+------------------+
| Router: softmax( |
|   W_router * x ) |  -> selects top-k experts (k=1 for Llama 4)
+------------------+
      |
      v  (only activated expert processes the token)
+------+------+------+-----+
| Exp1 | Exp2 | Exp3 | ... |  128 experts (Maverick)
+------+------+------+-----+
      |
      v
+------------------+
| Shared Expert    |  (always active, processes every token)
+------------------+
      |
      v
  weighted sum -> output
```

**Neuroscience parallel**: The router acts like the thalamus -- a gateway that routes information to specialized cortical regions (experts). The shared expert acts like the reticular formation -- always-on baseline processing.

---

## 2. Neuroscience-Inspired Architecture Modifications

### 2.1 Synaptic Weight Modulation

**Concept**: Add learnable scale factors to attention weights that act like synaptic strengths -- multiplicative gating per attention head.

#### Standard Attention
```
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V
```

#### Synaptically Modulated Attention
```
// Per-head learnable synaptic strength: alpha_h (scalar, initialized to 1.0)
// Per-head learnable synaptic gate:     g_h    (vector of dim d_k)

Attention_h(Q, K, V) = alpha_h * softmax((Q * g_h) K^T / sqrt(d_k)) * V

// Or more ambitiously, a full synaptic modulation matrix:
S_h = sigmoid(W_syn * [Q; K])   // context-dependent synaptic strengths
A_h = softmax(QK^T / sqrt(d_k))
Attention_h(Q, K, V) = (A_h * S_h) * V    // element-wise modulation
```

**Biological basis**: In the brain, synaptic strength varies per connection and is modulated by neuromodulators (dopamine, serotonin, acetylcholine). A synapse does not simply transmit; it amplifies or attenuates based on learned associations and current neuromodulatory state.

**Implementation approaches**:

**(a) Static per-head scaling (simplest)**
```python
class SynapticAttention(nn.Module):
    def __init__(self, num_heads, head_dim):
        self.alpha = nn.Parameter(torch.ones(num_heads))  # learnable per-head scale

    def forward(self, Q, K, V):
        attn = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.head_dim)
        attn = F.softmax(attn, dim=-1)
        attn = attn * self.alpha.view(1, -1, 1, 1)  # scale per head
        return torch.matmul(attn, V)
```

**(b) Context-dependent modulation (richer)**
```python
class NeuromodulatedAttention(nn.Module):
    def __init__(self, hidden_dim, num_heads):
        # Neuromodulator: maps global context to per-head modulation
        self.modulator = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim // 4),
            nn.SiLU(),
            nn.Linear(hidden_dim // 4, num_heads),
            nn.Sigmoid()
        )

    def forward(self, Q, K, V, context):
        modulation = self.modulator(context.mean(dim=1))  # [batch, num_heads]
        attn = standard_attention(Q, K, V)
        return attn * modulation.unsqueeze(-1).unsqueeze(-1)
```

**(c) Full synaptic modulation matrix**
```python
# For each (query_pos, key_pos) pair, learn a modulation strength
# This is O(n^2) in sequence length -- only feasible for local attention windows
class SynapticMatrix(nn.Module):
    def __init__(self, head_dim):
        self.W_syn = nn.Linear(2 * head_dim, 1)

    def forward(self, Q, K):
        # Q: [batch, heads, seq_q, d], K: [batch, heads, seq_k, d]
        # Compute pairwise synaptic strengths
        Q_exp = Q.unsqueeze(-2).expand(-1, -1, -1, K.size(-2), -1)
        K_exp = K.unsqueeze(-3).expand(-1, -1, Q.size(-2), -1, -1)
        pairs = torch.cat([Q_exp, K_exp], dim=-1)
        return torch.sigmoid(self.W_syn(pairs).squeeze(-1))
```

**Feasibility**: **(a)** is Adapter/LoRA compatible -- just add scalar parameters. **(b)** requires a small additional network but can be added as an adapter. **(c)** requires architectural change and retraining.

### 2.2 Cortical Columns via Grouped Attention

**Concept**: GQA already groups query heads into clusters sharing KV projections. Can we push this further so each group specializes as a "functional column"?

In neuroscience, cortical columns are vertical units of ~100 neurons spanning all 6 cortical layers, each specialized for a particular feature (orientation, frequency, spatial location). Different columns process different aspects of the same input in parallel.

#### Current GQA (Llama 3.1 8B: 4 queries per KV group, 8 groups)

```
Group 0: Q0, Q1, Q2, Q3  --> KV0   (all see same keys/values)
Group 1: Q4, Q5, Q6, Q7  --> KV1
...
Group 7: Q28-Q31          --> KV7
```

#### Proposed: Cortical Column Attention (CCA)

**Enhancement 1: Per-group specialization loss**
```
L_specialization = -sum_g sum_{g' != g} JS_divergence(A_g, A_{g'})

where A_g = average attention pattern of group g
```
This loss encourages groups to attend to different parts of the input (functional specialization).

**Enhancement 2: Cross-column lateral inhibition**
```python
class CorticalColumnAttention(nn.Module):
    def __init__(self, num_groups, heads_per_group, head_dim):
        # Lateral inhibition: each group can suppress others
        self.lateral_inhibition = nn.Parameter(
            torch.zeros(num_groups, num_groups)
        )  # off-diagonal = inhibition strength

    def forward(self, group_outputs):
        # group_outputs: [batch, num_groups, seq, dim]
        inhibition = torch.sigmoid(self.lateral_inhibition)
        inhibition.diagonal().fill_(1.0)  # no self-inhibition

        # Apply cross-column inhibition
        modulated = torch.einsum('bg..., gg->bg...', group_outputs, inhibition)
        return modulated
```

**Enhancement 3: Hierarchical column organization**
```
Layer 1-10:   Groups specialize in syntactic features (like V1/V2 in visual cortex)
Layer 11-20:  Groups specialize in semantic features (like V4/IT)
Layer 21-32:  Groups specialize in abstract reasoning (like PFC)

Implementation: Train with auxiliary losses that encourage feature type specialization
at different layer depths.
```

**Enhancement 4: Dynamic routing between columns (MoE-like)**
```
Instead of fixed group assignment, let tokens dynamically route to columns:

router_output = softmax(W_router * x)     # which column(s) should process this?
column_output_i = Column_i(x)             # each column processes independently
output = sum(router_output_i * column_output_i)
```

This merges MoE and GQA into a unified "cortical column" architecture -- essentially what Llama 4 does with its expert routing!

**Feasibility**: Enhancement 1 (specialization loss) can be applied during fine-tuning. Enhancement 2 (lateral inhibition) is Adapter-compatible. Enhancement 3 requires planned training from scratch. Enhancement 4 is essentially MoE, already proven.

### 2.3 Dual Process via Early Exit (System 1 / System 2)

**Concept**: Implement Kahneman's dual-process theory in the transformer:
- **System 1** (fast, intuitive): Exit early from the transformer for "easy" inputs (fewer layers)
- **System 2** (slow, deliberate): Use all layers for complex inputs

#### Architecture

```
Input x
  |
  v
[Layer 1] --> confidence_1 = Classifier(hidden_1)
  |           if confidence_1 > threshold: EXIT (System 1)
  v
[Layer 2] --> confidence_2 = Classifier(hidden_2)
  |           if confidence_2 > threshold: EXIT (System 1)
  v
  ...
[Layer L] --> always produces output (System 2 = full computation)
```

#### Key Papers on This Approach

1. **"Mixture of Depths"** (Raposo et al., 2024): Tokens dynamically skip layers based on a learned router. Unlike early exit, a token CAN skip middle layers and rejoin later.

2. **"ADEPT"** (Yoo et al., 2026): Adaptive Dynamic Early-Exit Process using MDP with preference-based rewards for token-level early exit.

3. **"Adaptive Thinking Using Dynamic Computation"** (ICLR 2025): Extends Mixture of Depth with fixed-point iteration and introspection.

4. **"Continuous Thought Machines"** (2025): Uses PonderNet/ACT-style halting for adaptive depth.

#### Implementation for Llama

```python
class DualProcessLlama(nn.Module):
    def __init__(self, base_model, exit_layers=[8, 16, 24]):
        self.layers = base_model.layers  # all 32 layers
        self.exit_classifiers = nn.ModuleDict({
            str(l): nn.Sequential(
                nn.Linear(hidden_dim, hidden_dim // 4),
                nn.SiLU(),
                nn.Linear(hidden_dim // 4, vocab_size)
            ) for l in exit_layers
        })
        self.confidence_head = nn.ModuleDict({
            str(l): nn.Linear(hidden_dim, 1) for l in exit_layers
        })

    def forward(self, x, threshold=0.9):
        for i, layer in enumerate(self.layers):
            x = layer(x)

            if str(i) in self.exit_classifiers:
                confidence = torch.sigmoid(self.confidence_head[str(i)](x))

                if confidence.mean() > threshold:  # System 1: exit early
                    return self.exit_classifiers[str(i)](x), i, "system1"

        return self.lm_head(x), len(self.layers), "system2"
```

**Training Strategy**:
```
L_total = L_final + sum_l (gamma^(L-l) * L_exit_l) + lambda * L_compute

where:
  L_final     = standard next-token prediction loss at final layer
  L_exit_l    = next-token prediction loss at exit point l
  gamma       = discount factor (earlier exits get less weight)
  L_compute   = penalty for using more layers (encourages early exit)
```

**Neuroscience connection**: This maps to how the brain handles routine vs. novel situations:
- Routine tasks (typing, walking) use subcortical/early cortical circuits (System 1)
- Novel/complex tasks recruit prefrontal cortex (PFC) and require sustained attention (System 2)
- The "confidence signal" maps to the anterior cingulate cortex (ACC) monitoring for conflict/uncertainty

**Feasibility**: The exit classifiers can be trained as adapters without modifying the base model. The base model layers remain frozen; only exit heads are trained. This is **Adapter-compatible** and has been demonstrated in multiple papers.

### 2.4 Prediction Error Signal

**Concept**: Augment the standard next-token prediction loss with a neuroscience-inspired prediction error (surprise) signal.

In the brain, prediction error is the difference between expected and actual sensory input. It's computed by:
- **Top-down predictions** from higher cortical areas
- **Bottom-up sensory signals** from lower areas
- **Error neurons** that compute the mismatch

#### Architecture: Predictive Coding Transformer

```
Standard Transformer:
  Layer L predicts nothing; only the final output predicts next token.

Predictive Coding Transformer:
  Each layer generates a prediction of its own input from the layer above.

Layer L (top):       h_L        --> predicts h_{L-1}
                      |
                   error_{L-1} = h_{L-1} - predict_{L}(h_L)
                      |
Layer L-1:        h_{L-1}      --> predicts h_{L-2}
                      |
                   error_{L-2} = h_{L-2} - predict_{L-1}(h_{L-1})
                      ...
Layer 1:           h_1
```

#### Implementation

```python
class PredictiveCodingLayer(nn.Module):
    def __init__(self, hidden_dim):
        self.standard_layer = LlamaDecoderLayer(...)
        # Prediction head: predict the representation of the layer below
        self.predictor = nn.Linear(hidden_dim, hidden_dim)

    def forward(self, x, prediction_from_above=None):
        h = self.standard_layer(x)

        # Compute prediction error if we have a top-down prediction
        if prediction_from_above is not None:
            error = x - prediction_from_above  # bottom-up minus top-down
            surprise = torch.norm(error, dim=-1).mean()
        else:
            surprise = 0.0

        # Generate prediction for the layer below
        prediction = self.predictor(h)

        return h, prediction, surprise
```

#### Loss Function with Prediction Error

```
L_total = L_next_token                          # standard causal LM loss
        + alpha * sum_l ||h_l - predict_{l+1}(h_{l+1})||^2   # prediction error
        + beta * L_contrastive                   # contrastive: correct next token
                                                 # vs. random tokens (CPC-style)
        + gamma * L_surprise_calibration         # are prediction errors well-calibrated?
```

**Contrastive Predictive Coding (CPC) component**:
```
L_CPC = -log(exp(f(h_t, x_{t+k})) / sum_j exp(f(h_t, x_j)))

where:
  h_t = hidden state at time t
  x_{t+k} = actual future token k steps ahead
  x_j = negative samples (random tokens)
  f = bilinear scoring function
```

**Paper**: "Learning Transformer-based World Models with Contrastive Predictive Coding" (2025, arXiv:2503.04416)

**Paper**: "Predictive coding algorithms induce brain-like responses in artificial neural networks" (PLOS Complex Systems, 2025)

**Feasibility**: The prediction heads can be added as **adapters** and trained while keeping the base model frozen. The CPC loss can be applied during fine-tuning. The full predictive coding architecture (error signals propagating backward) would require **retraining from scratch**.

### 2.5 Hebbian Attention

**Concept**: Instead of (or in addition to) standard Q*K^T attention, add a Hebbian component that strengthens attention patterns that led to good outcomes.

**Hebb's Rule**: "Neurons that fire together, wire together." In attention terms: if attending to token j when predicting token i led to a correct prediction, strengthen the attention connection i->j.

#### Key Paper

**"Short-term Hebbian Learning Can Implement Transformer-like Attention"** (Elliasmith et al., 2024, PLOS Computational Biology)

This paper proved that a biophysical model of short-term Hebbian synaptic potentiation can implement the same computation as transformer self-attention. The "match-and-control principle": when axonal activity is synchronous with somatic activity, the synapse is briefly potentiated, allowing that axon to control downstream neural activity.

#### Key Paper #2

**"Enabling Robust In-Context Memory and Rapid Task Adaptation in Transformers with Hebbian and Gradient-Based Plasticity"** (2025, arXiv:2510.21908)

This paper directly adds Hebbian plasticity to transformer feed-forward layers as "fast weights" that update within a single forward pass.

#### Implementation

```python
class HebbianAttention(nn.Module):
    """Attention with a Hebbian learning component that accumulates
    within the forward pass (sequence-level plasticity)."""

    def __init__(self, head_dim, hebbian_lr=0.01):
        self.hebbian_lr = hebbian_lr
        # Hebbian weight matrix: accumulates co-activation patterns
        # Reset at the start of each sequence
        self.register_buffer('H', torch.zeros(head_dim, head_dim))

    def forward(self, Q, K, V, reset=False):
        if reset:
            self.H.zero_()

        # Standard attention
        A_standard = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(Q.size(-1))

        # Hebbian attention: based on accumulated co-activation
        A_hebbian = torch.matmul(Q, self.H @ K.transpose(-2, -1))

        # Combine
        A_combined = A_standard + self.hebbian_lr * A_hebbian
        A = F.softmax(A_combined, dim=-1)

        output = torch.matmul(A, V)

        # Hebbian update: strengthen connections between co-active tokens
        # delta_H = eta * (Q^T @ A @ K)  -- outer product of attended patterns
        with torch.no_grad():
            self.H += self.hebbian_lr * torch.matmul(Q.transpose(-2, -1),
                                                       torch.matmul(A, K))
            # Decay to prevent runaway potentiation
            self.H *= 0.99

        return output
```

#### More Sophisticated: Reward-Modulated Hebbian Attention

```python
class RewardModulatedHebbianAttention(nn.Module):
    """Only strengthen attention patterns that led to correct predictions."""

    def forward(self, Q, K, V, reward=None):
        A = standard_attention(Q, K, V)
        output = torch.matmul(A, V)

        if reward is not None:
            # Three-factor learning rule: pre * post * reward
            # pre = K (what was attended to)
            # post = Q (what was attending)
            # reward = scalar signal from prediction accuracy
            delta_H = reward * torch.matmul(Q.transpose(-2, -1),
                                              torch.matmul(A, K))
            self.H += self.hebbian_lr * delta_H

        return output
```

**Feasibility**: The simple Hebbian component (without reward) can be implemented at **inference time** as a running accumulator. The reward-modulated version requires fine-tuning. The full integration into the attention mechanism requires **retraining**.

### 2.6 Habituation in Attention

**Concept**: Attention weights that decay for repeated patterns -- attention adaptation. In neuroscience, habituation is the simplest form of learning: decreased response to a repeated stimulus.

#### Biological Basis
- Neurons in sensory cortex show **stimulus-specific adaptation (SSA)**: reduced firing for repeated stimuli
- This frees up computational resources for novel stimuli
- Habituation is not fatigue -- it's stimulus-specific and recovers with novel input

#### Implementation

```python
class HabituatingAttention(nn.Module):
    """Attention that habituates to repeated patterns."""

    def __init__(self, num_heads, max_seq_len, habituation_rate=0.1):
        # Habituation map: tracks how often each attention pattern appears
        self.register_buffer('pattern_count',
                             torch.zeros(num_heads, max_seq_len, max_seq_len))
        self.habituation_rate = habituation_rate
        self.recovery_rate = 0.01  # slow recovery when pattern not seen

    def forward(self, Q, K, V, step=None):
        # Standard attention scores
        A = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(Q.size(-1))
        A_soft = F.softmax(A, dim=-1)

        # Habituation: reduce attention for frequently-attended patterns
        habituation_factor = 1.0 / (1.0 + self.habituation_rate * self.pattern_count)
        A_habituated = A_soft * habituation_factor
        A_habituated = A_habituated / A_habituated.sum(dim=-1, keepdim=True)  # renormalize

        # Update pattern counts
        with torch.no_grad():
            self.pattern_count *= (1.0 - self.recovery_rate)  # decay old patterns
            self.pattern_count += (A_soft > 0.1).float()      # count strong attention

        return torch.matmul(A_habituated, V)
```

#### Simpler Approach: Exponential Decay of Attention to Seen Tokens

```python
class AttentionWithDecay(nn.Module):
    """Tokens that have been heavily attended to in previous layers
    get reduced attention in later layers."""

    def __init__(self, decay_rate=0.05):
        self.decay_rate = decay_rate

    def forward(self, Q, K, V, cumulative_attention=None):
        A = standard_attention_scores(Q, K)

        if cumulative_attention is not None:
            # Reduce attention to tokens that have already been heavily attended
            novelty_bonus = 1.0 / (1.0 + self.decay_rate * cumulative_attention)
            A = A * novelty_bonus
            A = A / A.sum(dim=-1, keepdim=True)

        output = torch.matmul(A, V)

        # Update cumulative attention for next layer
        new_cumulative = (cumulative_attention or 0) + A.detach()

        return output, new_cumulative
```

**Connection to existing work**: Llama 4's NoPE layers (no positional encoding every 4th layer) can be seen as a form of "attention reset" -- removing the positional bias allows the model to "re-attend" without habitual local patterns.

**Feasibility**: The exponential decay approach can be implemented at **inference time** with no training needed. The learned habituation parameters would require **fine-tuning**. This is particularly interesting because it could help with repetition problems in generation -- the model would naturally attend less to recently-used tokens.

### 2.7 Population Coding in Attention

**Concept**: In neuroscience, population coding means that information is encoded in the collective activity of many neurons, not any single neuron. The "population vector" (weighted sum of preferred directions) predicts movement direction more accurately than any single neuron.

Applied to attention heads: instead of concatenating head outputs, use a **voting mechanism** where heads contribute to a population vector.

#### Standard Multi-Head Attention Output

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) * W_o
```
This is a simple concatenation + linear projection. Each head contributes equally.

#### Population-Coded Attention

```python
class PopulationCodedAttention(nn.Module):
    """Each head has a 'preferred direction' (feature space preference).
    The population vector is the confidence-weighted sum of head outputs."""

    def __init__(self, num_heads, head_dim, hidden_dim):
        # Each head's preferred direction (learned)
        self.preferred_direction = nn.Parameter(
            torch.randn(num_heads, head_dim)
        )
        # Tuning curve width (how selective each head is)
        self.tuning_width = nn.Parameter(torch.ones(num_heads))

        # Output projection
        self.W_o = nn.Linear(head_dim, hidden_dim)

    def forward(self, head_outputs, head_contexts):
        """
        head_outputs: [batch, num_heads, seq, head_dim]
        head_contexts: [batch, num_heads, seq, head_dim] (what each head attended to)
        """
        # Compute each head's "activation" (tuning curve response)
        # How well does the input match this head's preference?
        similarity = torch.cosine_similarity(
            head_contexts,
            self.preferred_direction.unsqueeze(0).unsqueeze(2),
            dim=-1
        )  # [batch, num_heads, seq]

        # Tuning curve: Gaussian response
        activation = torch.exp(-similarity**2 / (2 * self.tuning_width.unsqueeze(0).unsqueeze(-1)**2))

        # Population vector: activation-weighted sum of head outputs
        # (NOT concatenation -- weighted voting)
        population_vector = torch.sum(
            activation.unsqueeze(-1) * head_outputs,
            dim=1
        )  # [batch, seq, head_dim]

        return self.W_o(population_vector)
```

#### Alternative: Confidence-Weighted Voting

```python
class VotingAttention(nn.Module):
    """Each head votes on the output, weighted by its confidence."""

    def __init__(self, num_heads, head_dim):
        # Confidence estimator per head
        self.confidence = nn.ModuleList([
            nn.Linear(head_dim, 1) for _ in range(num_heads)
        ])

    def forward(self, head_outputs):
        # head_outputs: [batch, num_heads, seq, head_dim]
        confidences = torch.stack([
            torch.sigmoid(self.confidence[h](head_outputs[:, h]))
            for h in range(self.num_heads)
        ], dim=1)  # [batch, num_heads, seq, 1]

        # Normalize confidences
        confidences = confidences / confidences.sum(dim=1, keepdim=True)

        # Weighted vote
        output = (confidences * head_outputs).sum(dim=1)
        return output
```

**Connection to MoE**: The expert router in Llama 4 is already a form of population coding! Each expert "votes" on the output, weighted by the router's confidence. The shared expert provides a baseline "population activity."

**Feasibility**: Confidence-weighted voting can be added as an **adapter** layer (replacing or augmenting the output projection W_o). The full population coding model with tuning curves would require **fine-tuning** or **retraining**.

---

## 3. Implementation Feasibility Matrix

```
+------------------------------+-------------------+-----------+------------------+
| Modification                 | Category          | New Params| Estimated Cost   |
+------------------------------+-------------------+-----------+------------------+
| Static synaptic scaling      | Adapter/LoRA      | h scalars | Very cheap       |
| (per-head alpha)             |                   | (~128)    | (hours on 1 GPU) |
+------------------------------+-------------------+-----------+------------------+
| Context-dependent modulation | Adapter/LoRA      | ~1M       | Moderate         |
| (neuromodulator network)     |                   |           | (days on 8 GPU)  |
+------------------------------+-------------------+-----------+------------------+
| Full synaptic modulation     | Full Retrain      | O(n^2)    | Very expensive   |
| matrix                       |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Cortical column specializ.   | Fine-tune loss    | 0         | Moderate         |
| loss (per-group divergence)  |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Lateral inhibition between   | Adapter           | g*g       | Cheap            |
| GQA groups                   |                   | (~64)     |                  |
+------------------------------+-------------------+-----------+------------------+
| Dynamic column routing       | Full Retrain      | ~10M+     | Very expensive   |
| (MoE-style)                  | (already in L4)   |           | (= building MoE) |
+------------------------------+-------------------+-----------+------------------+
| Early exit classifiers       | Adapter           | ~10M per  | Moderate         |
| (System 1/2 dual process)    |                   | exit point|                  |
+------------------------------+-------------------+-----------+------------------+
| Mixture of Depths routing    | Full Retrain      | ~1M per   | Expensive        |
|                              |                   | layer     |                  |
+------------------------------+-------------------+-----------+------------------+
| Predictive coding heads      | Adapter           | d*d per   | Moderate         |
| (per-layer prediction)       |                   | layer     |                  |
+------------------------------+-------------------+-----------+------------------+
| Full predictive coding       | Full Retrain      | 2x params | Very expensive   |
| (bidirectional error flow)   |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| CPC contrastive loss         | Fine-tune loss    | ~1M       | Moderate         |
|                              |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Simple Hebbian accumulator   | Inference-Time    | 0 (buffer)| Free (runtime)   |
| (no reward modulation)       |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Reward-modulated Hebbian     | Fine-tune         | ~1M       | Moderate         |
|                              |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Full Hebbian attention       | Full Retrain      | +50%      | Very expensive   |
| (replaces Q*K^T)             |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Attention decay / habituation| Inference-Time    | 0 (buffer)| Free (runtime)   |
| (exponential decay)          |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Learned habituation params   | Adapter/LoRA      | h params  | Cheap            |
|                              |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Confidence-weighted voting   | Adapter           | h*d       | Cheap to         |
| (population coding lite)     |                   |           | moderate         |
+------------------------------+-------------------+-----------+------------------+
| Full population coding       | Full Retrain      | ~10M      | Expensive        |
| (tuning curves, pref dirs)   |                   |           |                  |
+------------------------------+-------------------+-----------+------------------+
| Adaptive temperature per head| Inference-Time /  | h or 0    | Free to cheap    |
|                              | Adapter           |           |                  |
+------------------------------+-------------------+-----------+------------------+
```

### Summary by Category

**Can Be Done at Inference Time (No Training):**
1. Simple Hebbian accumulator (running co-activation matrix)
2. Attention habituation (exponential decay for repeated patterns)
3. Adaptive temperature per head (learned or heuristic)
4. Population-style voting with fixed confidence heuristics

**Can Be Done with Adapters/LoRA (Cheap Fine-Tuning):**
1. Static synaptic scaling (per-head alpha)
2. Context-dependent neuromodulation network
3. Lateral inhibition between GQA groups
4. Early exit classifiers (System 1/2)
5. Per-layer prediction heads (predictive coding lite)
6. Confidence-weighted head voting
7. Learned habituation parameters

**Requires Full Retraining:**
1. Full synaptic modulation matrix
2. Dynamic column routing (MoE-style)
3. Full predictive coding with bidirectional error flow
4. Mixture of Depths
5. Full Hebbian attention (replacing Q*K^T)
6. Full population coding with tuning curves

**Not Feasible / Impractical:**
- None of the proposed modifications are fundamentally infeasible. All have theoretical backing and some have been demonstrated in papers. The question is always cost vs. benefit.

---

## 4. Research Papers and Prior Art

### 4.1 Biologically-Inspired Transformers

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| "Building Transformers from Neurons and Astrocytes" (Kozachkov et al.) | 2023 | PNAS paper showing astrocyte biology can implement self-attention biophysically |
| "Short-term Hebbian Learning Can Implement Transformer-like Attention" (Elliasmith et al.) | 2024 | PLOS Comp Bio: proved Hebbian plasticity computes attention |
| "Enabling Robust In-Context Memory with Hebbian and Gradient-Based Plasticity" | 2025 | Added Hebbian fast-weights to transformer FFN layers |
| "Neural Attention: Enhanced Expressive Power" (arXiv:2502.17206) | 2025 | Replaced dot products in attention with feed-forward networks |
| "Beyond Markov: Transformers, Memory, and Attention" | 2025 | Connected transformers to neuroscience theories of memory and prediction |
| "From Transformer to Biology: Hierarchical Model for Attention" | 2024 | Hierarchical attention model inspired by brain's attentional hierarchy |
| "Titans: Learning to Memorize at Test Time" (Google) | 2024 | Brain-inspired memory hierarchy (short-term, working, long-term) in neural nets |

### 4.2 Mixture of Experts as Cortical Columns

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| "Switch Transformers" (Fedus et al., Google) | 2022 | Simplified MoE routing, one expert per token |
| "Mixture-of-Depths" (Raposo et al.) | 2024 | Tokens skip layers dynamically |
| "Mixture-of-Recursions" (NeurIPS 2025) | 2025 | Shared layer stack with adaptive recursion depth per token |
| "On the Spatial Structure of MoE in Transformers" (arXiv:2504.04444) | 2025 | Analyzed how experts organize spatially |
| Llama 4 (Meta) | 2025 | 128-expert MoE with shared expert, interleaved dense/MoE layers |
| "GShard" (Lepikhin et al., Google) | 2021 | Scaling MoE to 600B parameters |

**The MoE-cortical-column analogy is strong:**
- Cortical columns: ~150,000 functional units, each specialized, operating in parallel
- MoE experts: 128 (Llama 4), each a mini-FFN, dynamically routed
- The router = thalamic gating
- The shared expert = baseline cortical activation
- Expert specialization emerges naturally from training (like cortical maps emerge from experience)

### 4.3 Sparse Attention as Attentional Filtering

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| "Generating Long Sequences with Sparse Transformers" (Child et al.) | 2019 | Original sparse attention patterns |
| "BigBird" (Zaheer et al.) | 2020 | Random + local + global sparse attention |
| "Longformer" (Beltagy et al.) | 2020 | Sliding window + global attention |
| "Sparser is Faster and Less is More" (arXiv:2406.16747) | 2024 | Top-k sparse attention with differentiable selection |
| Llama 4 iRoPE chunked attention | 2025 | Local chunked attention + periodic global attention |

**Biological parallel**: The brain's attentional system selectively processes stimuli:
- **Spotlight attention** = local/sliding window attention
- **Divided attention** = global attention tokens
- **Attentional blink** = computational cost of switching attention focus
- **Inhibition of return** = habituation/decay for attended locations

### 4.4 Adaptive Computation Time as Dual Process

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| "Adaptive Computation Time for RNNs" (Graves) | 2016 | Original ACT: learn when to stop computing |
| "PonderNet" (Banino et al., DeepMind) | 2021 | Probabilistic halting for adaptive depth |
| "Universal Transformers" (Dehghani et al.) | 2019 | Shared-weight transformers with ACT |
| "Test-time Computing: System-1 to System-2" (arXiv:2501.02497) | 2025 | Comprehensive survey of adaptive test-time compute |
| "ADEPT" (arXiv:2601.03700) | 2026 | MDP-based token-level early exit |
| "Change of Thought: Adaptive Test-Time Computation" | 2025 | SELF-Transformer: iteratively refines attention to fixed point |
| "Continuous Thought Machines" | 2025 | Neuroscience-inspired continuous-time adaptive computation |

### 4.5 Learned Temperature and Sampling

| Paper | Year | Key Contribution |
|-------|------|-----------------|
| "Hot or Cold? Adaptive Temperature Sampling" (AAAI 2024) | 2024 | Dynamic temperature based on token difficulty |
| "Adaptive Decoding via Latent Preference Optimization" | 2024 | Learned per-token temperature from hidden states |
| "Calibration Attention" (arXiv:2508.08547) | 2025 | Per-instance temperature from CLS token in ViT |
| "Optimizing Temperature for Multi-Sample Inference" | 2025 | Optimal temperature varies by model and task |
| Llama 4 inference-time temperature scaling | 2025 | Temperature scaling in NoPE layers for length generalization |

---

## 5. Custom Training Objectives

### 5.1 Brain-Like Loss Function Design

A "neuroscience-complete" loss function would combine multiple objectives that mirror brain learning signals:

```
L_total = L_prediction          # primary: predict next token (like cortical prediction)
        + alpha * L_surprise     # prediction error signal (like ACC/dopamine)
        + beta * L_specialization # encourage functional specialization (like cortical maps)
        + gamma * L_calibration   # confidence should match accuracy (like metacognition)
        + delta * L_efficiency    # minimize computation (like metabolic cost)
        + epsilon * L_coherence   # maintain goal consistency (like PFC sustained activity)
```

#### 5.1.1 Prediction Loss (Standard)
```
L_prediction = -sum_t log P(x_{t+1} | x_{1:t})
```
This is the standard causal language modeling loss. Biologically, this corresponds to the brain's core function: predict the next sensory input.

#### 5.1.2 Surprise / Prediction Error Loss
```
L_surprise = sum_l ||h_l - f_l(h_{l+1})||^2    # inter-layer prediction error

// Plus: contrastive surprise
L_CPC = -sum_t log [ exp(sim(h_t, e_{t+k})) / sum_j exp(sim(h_t, e_j)) ]

where:
  h_t = hidden state at position t
  e_{t+k} = embedding of token k steps ahead
  e_j = embeddings of random negative samples
  sim = bilinear similarity function
```

**Biological basis**: Dopaminergic neurons in the VTA encode reward prediction error. Cortical prediction error neurons compute mismatch between top-down predictions and bottom-up input. This signal drives learning: large errors cause large weight updates (surprise = learning opportunity).

#### 5.1.3 Specialization Loss
```
L_specialization = -sum_g sum_{g' != g} D_JS(A_g || A_{g'})

where:
  A_g = average attention distribution of GQA group g
  D_JS = Jensen-Shannon divergence
```

Encourages different attention head groups to attend to different things, mirroring cortical column specialization.

**Alternative: Expert Specialization (for MoE)**
```
L_expert_spec = -H(p(expert | token_type))

where H is entropy and token_type is derived from
syntactic/semantic clustering of tokens
```
Low entropy = high specialization = each expert handles specific token types.

#### 5.1.4 Calibration Loss
```
L_calibration = sum_b (confidence_b - accuracy_b)^2

where:
  confidence_b = average predicted probability in confidence bin b
  accuracy_b = fraction of correct predictions in bin b
```

This is Expected Calibration Error (ECE) as a differentiable loss. Biologically, this corresponds to **metacognition** -- the brain's ability to assess its own uncertainty. Well-calibrated organisms make better decisions.

#### 5.1.5 Efficiency Loss (Metabolic Cost)
```
L_efficiency = sum_l sum_t C(l, t)

where C(l, t) = {
  0       if token t skips layer l (early exit / MoD)
  c_l     if token t passes through layer l
}
```

The brain consumes ~20% of body energy despite being ~2% of body weight. Metabolic pressure drives efficient neural coding -- neurons that are not needed are suppressed. This loss directly maps to the "mixture of depths" and early exit mechanisms.

#### 5.1.6 Coherence Loss (Goal Tracking)
```
L_coherence = -cosine_similarity(h_t^L, goal_embedding)

where:
  h_t^L = final layer hidden state at current position
  goal_embedding = running representation of the task/goal
```

The prefrontal cortex maintains goal representations via sustained neural activity. This loss ensures the model stays on track, analogous to corteX's own goal tracker.

### 5.2 Multi-Objective Training Strategy

Training with all objectives simultaneously requires careful balancing:

```
Phase 1 (Pretraining):
  L = L_prediction + 0.01 * L_surprise + 0.001 * L_specialization
  (Focus on language modeling, gentle push toward brain-like properties)

Phase 2 (Continued Pretraining):
  L = L_prediction + 0.1 * L_surprise + 0.01 * L_specialization
      + 0.01 * L_calibration
  (Increase brain-like objectives as model stabilizes)

Phase 3 (Fine-Tuning):
  L = L_prediction + 0.1 * L_surprise + 0.1 * L_specialization
      + 0.1 * L_calibration + 0.05 * L_efficiency + 0.05 * L_coherence
  (Full multi-objective with task-specific coherence)
```

### 5.3 Reinforcement Learning from Brain-Inspired Reward Signals

Instead of RLHF (human feedback), use **RLBF (Brain-inspired Feedback)**:

```
Reward_brain(trajectory) =
    w1 * R_prediction_accuracy    # did the model predict correctly?
  + w2 * R_surprise_appropriate   # was surprise calibrated? (surprised by surprising things)
  + w3 * R_efficiency             # did the model use minimal computation?
  + w4 * R_coherence              # did the model stay on goal?
  + w5 * R_novelty                # did the model explore new patterns?
  - w6 * R_repetition             # penalize repetitive loops (habituation signal)
```

**Training data generation for brain-like reasoning:**

1. **Prediction tasks**: Given context, predict multiple future tokens. Score: accuracy at different horizons (like temporal prediction in the brain).

2. **Anomaly detection**: Insert anomalies into text, reward the model for detecting them (prediction error signal).

3. **Multi-step planning**: Tasks that require maintaining a goal over many steps, with distractors. Reward coherence.

4. **Dual-process tasks**: Mix "easy" tasks (should be solved quickly, System 1) with "hard" tasks (should use more computation, System 2). Reward efficiency.

5. **Transfer/generalization**: Present pattern in one domain, test generalization to another. Reward abstract pattern matching (like cortical generalization).

### 5.4 Connection to corteX SDK

These training objectives directly parallel corteX's existing brain engine:

```
corteX Module              <-->    Training Objective
-----------------------------------------------------------
WeightEngine               <-->    Synaptic scaling (L_prediction with per-head alpha)
GoalTracker                <-->    Coherence loss (L_coherence)
PredictionEngine           <-->    Surprise loss (L_surprise, L_CPC)
FeedbackEngine             <-->    RLBF reward signals
PlasticityEngine           <-->    Hebbian attention updates
AdaptationEngine           <-->    Habituation decay
PopulationEngine           <-->    Population-coded voting
CalibrationEngine          <-->    Calibration loss (L_calibration)
CorticalColumns            <-->    Specialization loss (L_specialization)
ResourceMap                <-->    Efficiency loss (L_efficiency)
AttentionalFilter          <-->    Sparse attention / habituation
DualProcess (Orchestrator) <-->    Early exit / adaptive depth
```

**Insight**: corteX already implements these neuroscience principles at the *agent orchestration level*. The research in this document explores implementing them at the *model architecture level*. A fully aligned system would have brain-inspired patterns at both levels -- the model itself thinks in brain-like ways, and the orchestration layer adds additional brain-like coordination.

---

## 6. Synthesis: A Neuroscience-Enhanced Llama Architecture

### 6.1 Proposed Architecture: "NeuroLlama"

Combining the most feasible modifications into a coherent architecture:

```
                         NeuroLlama Architecture
                         ======================

Input Tokens --> Embedding + RoPE
                     |
    +----------------+----------------+
    |                                 |
    v                                 v
[SYSTEM 1 PATH]               [SYSTEM 2 PATH]
(Early Exit at Layer 8)        (Full 32 Layers)
    |                                 |
    |     +---------------------------+
    |     |
    v     v
+---Layer Block (x32)---+
|                        |
|  RMSNorm               |
|  |                     |
|  Cortical Column GQA   |  <-- Per-group specialization
|  |  + Synaptic alpha   |  <-- Learnable per-head scale
|  |  + Hebbian accum.   |  <-- Running co-activation matrix
|  |  + Habituation decay|  <-- Suppress repeated patterns
|  |                     |
|  + Residual            |
|  |                     |
|  RMSNorm               |
|  |                     |
|  SwiGLU FFN            |
|  |  + Prediction Head  |  <-- Predict next layer's input
|  |                     |
|  + Residual            |
|  |                     |
|  Confidence Estimator  |  <-- Can we exit early?
|  |                     |
+------------------------+
         |
         v
    Population-Coded Output  <-- Confidence-weighted head voting
         |
         v
    Final LM Head
```

### 6.2 What Can Be Done TODAY (Inference-Time, No Training)

These modifications can be applied to any existing Llama model immediately:

1. **Hebbian attention accumulator**: Track co-activation patterns across the forward pass, use them to bias attention
2. **Attention habituation**: Apply exponential decay to tokens that received heavy attention in previous layers
3. **Adaptive per-head temperature**: Scale attention logits differently per head based on entropy heuristics
4. **Population-style output weighting**: Weight head contributions by their attention entropy (confident heads get more weight)

These are essentially what corteX already does at the orchestration level -- but they could also be applied inside the model itself during inference, as a "wrapper" around standard Llama inference.

### 6.3 What Should Be Done Next (Adapter Training)

With a modest compute budget (8 GPUs for a few days):

1. **Train early exit classifiers** at layers 8, 16, 24 for System 1/2 dual processing
2. **Train per-head synaptic scaling factors** (alpha_h) to learn optimal head importance
3. **Train lateral inhibition matrix** between GQA groups for column specialization
4. **Train prediction heads** at each layer for predictive coding
5. **Fine-tune with specialization loss** to encourage GQA group differentiation

### 6.4 What Requires a Major Investment (Full Retraining)

For a research lab with serious compute:

1. **Full predictive coding transformer**: Every layer predicts its input from the layer above, error signals flow backward
2. **Mixture of Depths + Early Exit**: Dynamic layer skipping with System 1/2 routing
3. **Hebbian attention replacing Q*K^T**: Replace standard attention with biologically-plausible Hebbian computation
4. **Multi-objective pretraining**: Train from scratch with L_prediction + L_surprise + L_specialization + L_calibration + L_efficiency

### 6.5 Strategic Recommendation for corteX

The most impactful approach for corteX is a **two-level brain-inspired architecture**:

```
Level 1: Model Level (inside the transformer)
  - Inference-time Hebbian accumulation
  - Inference-time habituation
  - Adapter-trained early exit (System 1/2)
  - Adapter-trained prediction heads (surprise signal)

Level 2: Orchestration Level (corteX engine)
  - WeightEngine modulating tool/LLM selection
  - GoalTracker maintaining task coherence
  - PredictionEngine anticipating next steps
  - FeedbackEngine learning from outcomes
  - PlasticityEngine adapting over time
```

This creates a **doubly brain-inspired system**: the model itself has brain-like properties, and the orchestration layer adds metacognitive brain-like coordination. This is analogous to how the brain operates at both the **neural circuit level** (local computation) and the **brain network level** (global coordination).

---

## Sources

### Llama Architecture
- [Meta AI Blog: Llama 4 Multimodal Intelligence](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)
- [Meta AI Blog: Llama 3.1 Introduction](https://ai.meta.com/blog/meta-llama-3-1/)
- [Llama 3 Architecture Overview - EmergentMind](https://www.emergentmind.com/topics/llama-3-architecture)
- [Llama 4 Architecture Deconstructed: MoE, iRoPE - Medium](https://medium.com/@mandeep0405/llama-4s-architecture-deconstructed-moe-irope-and-early-fusion-explained-e58eb9403067)
- [The Llama 3 Herd of Models - arXiv](https://ar5iv.labs.arxiv.org/html/2407.21783)
- [Llama 4 HuggingFace Documentation](https://huggingface.co/docs/transformers/en/model_doc/llama4)
- [Welcome Llama 4 Maverick & Scout on HuggingFace](https://huggingface.co/blog/llama4-release)
- [Llama 4 Deep Dive - Cameron Wolfe](https://cameronrwolfe.substack.com/p/llama-4)
- [Deep Dive into LlaMA 3 by Hand - Towards Data Science](https://towardsdatascience.com/deep-dive-into-llama-3-by-hand-%EF%B8%8F-6c6b23dc92b2/)
- [HuggingFace Llama 3.1 Blog](https://huggingface.co/blog/llama31)

### Transformer Components
- [RoFormer: Rotary Position Embedding - arXiv](https://arxiv.org/pdf/2104.09864)
- [Inside RoPE: Rotary Magic - LearnOpenCV](https://learnopencv.com/rope-position-embeddings/)
- [Rotary Embeddings - EleutherAI](https://blog.eleuther.ai/rotary-embeddings/)
- [GLU Variants Improve Transformer - Shazeer 2020 - arXiv](https://arxiv.org/pdf/2002.05202)
- [SwiGLU Activation Function - EmergentMind](https://www.emergentmind.com/topics/swiglu-activation)
- [RMSNorm - arXiv](https://arxiv.org/abs/1910.07467)
- [RMSNorm and LayerNorm - Machine Learning Mastery](https://machinelearningmastery.com/layernorm-and-rms-norm-in-transformer-models/)
- [KV Caching Explained - HuggingFace](https://huggingface.co/blog/not-lain/kv-caching)
- [GQA: Grouped Query Attention - PyImageSearch](https://pyimagesearch.com/2025/10/06/introduction-to-kv-cache-optimization-using-grouped-query-attention/)

### Neuroscience-Inspired Architectures
- [Building Transformers from Neurons and Astrocytes - PNAS](https://www.pnas.org/doi/10.1073/pnas.2219150120)
- [Short-term Hebbian Learning Can Implement Transformer-like Attention - PLOS Comp Bio](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1011843)
- [Hebbian and Gradient-Based Plasticity in Transformers - arXiv:2510.21908](https://arxiv.org/abs/2510.21908)
- [Neural Attention: Enhanced Expressive Power - arXiv:2502.17206](https://arxiv.org/abs/2502.17206)
- [Beyond Markov: Transformers, Memory, and Attention](https://www.tandfonline.com/doi/full/10.1080/17588928.2025.2484485)
- [Titans: Learning to Memorize at Test Time - Google](https://aipapersacademy.com/titans/)
- [Google Introduces Transformer 2.0 - Data Science Collective](https://medium.com/data-science-collective/google-introduces-transformer-2-0-with-a-neuroscience-inspired-architecture-cf7f0d8836a4)
- [From Transformer to Biology: Hierarchical Model for Attention](https://arxiv.org/html/2406.14100v2)

### Adaptive Computation and Dual Process
- [Mixture-of-Depths: Dynamic Compute Allocation - arXiv:2404.02258](https://arxiv.org/html/2404.02258v1)
- [Mixture-of-Recursions - NeurIPS 2025](https://github.com/raymin0223/mixture_of_recursions)
- [ADEPT: Adaptive Dynamic Early-Exit - arXiv:2601.03700](https://www.arxiv.org/pdf/2601.03700)
- [Test-time Computing: System-1 to System-2 - arXiv:2501.02497](https://arxiv.org/html/2501.02497v1)
- [Continuous Thought Machines](https://arxiv.org/html/2505.05522v4)
- [Adaptive Thinking Using Dynamic Computation - ICLR 2025](https://proceedings.iclr.cc/paper_files/paper/2025/file/955499a8e2860ed746717c1374224c43-Paper-Conference.pdf)

### Predictive Coding and Contrastive Learning
- [Learning Transformer World Models with CPC - arXiv:2503.04416](https://arxiv.org/abs/2503.04416)
- [Predictive Coding Algorithms Induce Brain-like Responses - PLOS](https://journals.plos.org/complexsystems/article?id=10.1371/journal.pcsy.0000076)
- [Predictive Coding: Theoretical and Experimental Review - arXiv:2107.12979](https://arxiv.org/abs/2107.12979)

### Sparse Attention and MoE
- [Sparser is Faster and Less is More - arXiv:2406.16747](https://arxiv.org/abs/2406.16747)
- [On the Spatial Structure of MoE in Transformers - arXiv:2504.04444](https://arxiv.org/abs/2504.04444)
- [Survey on Mixture of Experts](https://arxiv.org/html/2407.06204v2)
- [Mixture of Experts Explained - HuggingFace](https://huggingface.co/blog/moe)

### Adaptive Temperature and Sampling
- [Adaptive Decoding via Latent Preference Optimization](https://arxiv.org/html/2411.09661v1)
- [Calibration Attention: Instance-wise Temperature Scaling - arXiv:2508.08547](https://arxiv.org/html/2508.08547v1)
- [Hot or Cold? Adaptive Temperature Sampling - AAAI 2024](https://dl.acm.org/doi/10.1609/aaai.v38i1.27798)
- [Optimizing Temperature for Multi-Sample Inference](https://arxiv.org/html/2502.05234v2)
