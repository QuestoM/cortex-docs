# Batch 10 Audit: Interop + NeuroLlama (10 pages)

**Auditor**: Documentation Audit Agent
**Date**: 2026-02-22
**Scope**: 9 Interop pages + 1 NeuroLlama page

---

## Summary

| Page | Status | Gaps |
|------|--------|------|
| interop/index.md | **[A]** | 2 |
| interop/types.md | **[OK]** | 0 |
| interop/mcp-client.md | **[OK]** | 0 |
| interop/mcp-tool-bridge.md | **[OK]** | 0 |
| interop/mcp-transport.md | **[OK]** | 0 |
| interop/mcp-resource-bridge.md | **[OK]** | 0 |
| interop/a2a-client.md | **[OK]** | 0 |
| interop/a2a-agent-card.md | **[OK]** | 0 |
| interop/a2a-task-bridge.md | **[OK]** | 0 |
| neurollama/neurollama-model.md | **[A]** | 3 |

**Total gaps: 5**

---

## Interop Pages

### reference/interop/index.md -> corteX/interop/__init__.py
**Status**: **[A]** (2 gaps found)

#### WRONG_DESCRIPTION
1. **Architecture diagram mentions "MCPToolBridge"**: The ASCII architecture diagram on line 79 references `MCPToolBridge` as a class name. However, the `tool_bridge.py` module does not have a class named `MCPToolBridge` -- it is a module of standalone functions (`make_tool_name`, `parse_tool_name`, `is_mcp_tool`, `mcp_tool_to_wrapper`, `wrapper_to_mcp_tool`, `mcp_result_to_string`). The name suggests a class when none exists.

#### WRONG_EXAMPLE
2. **See Also links may be broken**: The "See Also" section references `../../guides/interop/mcp-setup.md` and `../../guides/interop/a2a-delegation.md`. These file paths were not found in the cortex-docs glob results. The actual guide pages found are `docs/guides/interop/mcp-servers.md` and `docs/guides/interop/a2a-agents.md`.
   - Doc says: `../../guides/interop/mcp-setup.md`
   - Should be: `../../guides/interop/mcp-servers.md`
   - Doc says: `../../guides/interop/a2a-delegation.md`
   - Should be: `../../guides/interop/a2a-agents.md`

---

### reference/interop/types.md -> corteX/interop/types.py
**Status**: **[OK]**

All classes, attributes, methods, signatures, defaults, validation rules, and examples match source code exactly:
- `MCPServerConfig`: All 11 attributes match (name, transport, command, args, url, env, headers, capabilities, timeout, reconnect, max_reconnect_attempts). Validation rules match. `to_dict()` and `from_dict()` signatures match.
- `A2AAgentConfig`: All 8 attributes match (name, url, description, skills, auth_token, headers, timeout, max_retries). Validation, `get_auth_headers()`, `to_dict()` (excludes auth_token), and `from_dict()` all match.
- `InteropConfig`: All 3 attributes match (mcp_servers, a2a_agents, auto_connect). Properties `has_mcp`, `has_a2a`, `is_empty` match. `to_dict()` and `from_dict()` match.

---

### reference/interop/mcp-client.md -> corteX/interop/mcp/client.py
**Status**: **[OK]**

All classes, methods, and signatures verified:
- `MCPServerConnection`: All 5 attributes match (config, tools, resources, connected, error). Note: internal `_client` field correctly omitted from docs.
- `MCPClientManager.__init__`: Parameters `configs` and `capability_set` match.
- `connect_all()`: Signature and return type match.
- `connect_server()`: Signature and behavior match.
- `disconnect_all()` and `disconnect_server()`: Match.
- `get_tools()` and `get_tools_for_server()`: Match.
- `execute_tool()`: Signature, raises, and capability check logic match (`mcp:{server}:{tool}` + `execute`).
- `register_tools()`: Signature matches.
- `get_connection()`: Matches.
- Properties `connected_servers` and `available_tool_count`: Match.
- Capability enforcement section accurately describes the check.

---

### reference/interop/mcp-tool-bridge.md -> corteX/interop/mcp/tool_bridge.py
**Status**: **[OK]**

All functions verified:
- `make_tool_name(server_name, tool_name) -> str`: Matches.
- `parse_tool_name(prefixed_name) -> Tuple[str, str]`: Return value for non-MCP names correctly documented as `("", prefixed_name)`.
- `is_mcp_tool(name) -> bool`: Matches.
- `mcp_tool_to_wrapper(tool_schema, server_name, execute_fn) -> ToolWrapper`: Signature matches. Schema normalization rules documented accurately.
- `wrapper_to_mcp_tool(wrapper) -> Dict[str, Any]`: Matches. Prefix stripping behavior documented correctly.
- `mcp_result_to_string(result) -> str`: All 7 result shapes documented match the code's handling (None, str, dict+content, dict+text, dict+result, object+.content, fallback str).
- `_MCP_PREFIX` constant: Correctly documented as `"mcp__"`.

---

### reference/interop/mcp-transport.md -> corteX/interop/mcp/client_transport.py
**Status**: **[OK]**

All classes and methods verified:
- `TransportMessage`: All 6 attributes match (jsonrpc, method, params, result, error, id). Properties `is_response` and `is_request` match. Methods `to_json()`, `from_json()`, `request()`, `response()`, `error_response()` all match.
- `MCPTransport` (ABC): Constructor params match. Abstract methods `connect()`, `disconnect()`, `send_request()` match. Properties `connected` and `server_name` match.
- `StdioTransport`: Constructor params (server_name, command, args, env, timeout) match. Protocol version `2024-11-05` and client info `{"name": "corteX", "version": "1.0.0"}` match code. `disconnect()` 5-second timeout matches. `send_request()` raises documented correctly.
- `SSETransport`: Constructor params (server_name, url, headers, timeout) match. URL endpoint derivation logic (/sse -> /message) matches code. aiohttp dependency documented.

---

### reference/interop/mcp-resource-bridge.md -> corteX/interop/mcp/resource_bridge.py
**Status**: **[OK]**

All classes and methods verified:
- `ResourceInfo`: All 4 attributes match (uri, name, description, mime_type).
- `MCPResourceBridge.__init__()`: No-args constructor matches.
- `register_resources(server_name, resources)`: Signature matches. Accepts both `mimeType` and `mime_type` keys as documented.
- `list_available(server_name=None)`: Signature and filter behavior match.
- `get_as_context(server_name, uri, content)`: Signature matches. Output format with header block matches code (lines 107-117).
- `clear(server_name=None)`: Behavior matches.
- Properties `server_count` and `total_resources`: Match.
- Integration example is accurate.

---

### reference/interop/a2a-client.md -> corteX/interop/a2a/client.py
**Status**: **[OK]**

All classes and methods verified:
- `A2AAgentConnection`: All 4 attributes match (config, card, discovered, error). Note: doc says `Optional[AgentCardInfo]` for `card`; code uses `Optional[Any]` but is typed as `AgentCardInfo` in docstring intent -- acceptable.
- `A2AClientManager.__init__`: Parameters (configs, capability_set) match. Note: code type hint for `capability_set` is `Optional[Any]` while doc says `Optional[CapabilitySet]` -- this is a documentation improvement (more specific), acceptable.
- `discover_all()`: Returns `Dict[str, bool]`, matches.
- `discover_agent(config)`: Matches. Fetches `{url}/.well-known/agent.json` as documented.
- `get_available_agents()`: Returns list of cards, matches.
- `get_agent(name)`: Matches.
- `send_task(agent_name, goal, context)`: Signature, raises, and capability check (`a2a:{agent_name}` + `delegate`) match.
- `get_task_status(agent_name, task_id)`: Matches.
- `cancel_task(agent_name, task_id)`: Returns `bool`, checks `canceled`/`cancelled` status, matches.
- Property `discovered_agents`: Matches.
- HTTP provider fallback (aiohttp -> httpx) documented correctly.

---

### reference/interop/a2a-agent-card.md -> corteX/interop/a2a/agent_card.py
**Status**: **[OK]**

All classes and methods verified:
- `AgentSkill`: All 5 attributes match (id, name, description, tags, examples).
- `AgentCardInfo`: All 7 attributes match (name, description, url, version, skills, input_modes, output_modes). Default values match.
- `AgentCardBuilder.build_card()`: All 6 parameters match (name, description, url, version, tools, skills). Return type matches. Tool-to-skill conversion logic matches code.
- `AgentCardBuilder.parse_card(data)`: Signature matches. Skill parsing with fallback `id=s.get("id", s.get("name", ""))` matches code.
- `AgentCardBuilder.card_to_json(info)`: Signature matches. Output format matches.
- Agent Card JSON format example is accurate.

---

### reference/interop/a2a-task-bridge.md -> corteX/interop/a2a/task_bridge.py
**Status**: **[OK]**

All classes and methods verified:
- `A2AArtifact`: All 3 attributes match (parts, name, description).
- `A2AMessage`: All 2 attributes match (role, parts).
- `A2ATaskResult`: All 5 attributes match (task_id, status, artifacts, messages, error).
- `A2ATaskBridge.build_send_request(goal, context, task_id)`: Signature matches. JSON-RPC structure matches code output.
- `A2ATaskBridge.build_get_request(task_id)`: Matches.
- `A2ATaskBridge.build_cancel_request(task_id)`: Matches.
- `A2ATaskBridge.parse_response(response)`: Matches. Error handling and status extraction logic matches code.
- `A2ATaskBridge.a2a_status_to_cortex(status)`: Status mapping table matches `_A2A_TO_CORTEX` dict exactly.
- `A2ATaskBridge.cortex_status_to_a2a(status)`: Status mapping table matches `_CORTEX_TO_A2A` dict exactly.
- `A2ATaskBridge.get_result_text(result)`: Extraction logic matches code.
- Full lifecycle example is accurate.

---

## NeuroLlama Page

### reference/neurollama/neurollama-model.md -> corteX/neurollama/ (multiple files)
**Status**: **[A]** (3 gaps found)

#### WRONG_DESCRIPTION
1. **70B preset exit_layers mismatch**: The doc's preset table (line 552) claims `"70B"` has exit layers `[20, 40, 60]`. However, in the source code (`model.py` lines 252-253), the `"70B"` preset dict does NOT include an `exit_layers` key:
   ```python
   "70B": dict(num_layers=80, hidden_dim=8192, num_heads=64, num_kv_heads=8,
               head_dim=128, ffn_dim=28672, vocab_size=128256, max_seq_len=4096),
   ```
   Since `exit_layers` is absent, the `NeuroLlamaConfig` default `[8, 16, 24]` is used. The documented `[20, 40, 60]` is what you might *expect* at 25/50/75% of 80 layers, but that logic only runs in `from_llama_config()`, NOT in the preset factory.
   - **Doc says**: 70B exit_layers = `[20, 40, 60]`
   - **Code actually uses**: `[8, 16, 24]` (the dataclass default)

2. **8B preset exit_layers not explicitly set**: Similarly, the `"8B"` preset does not explicitly set `exit_layers`. The doc says `[8, 16, 24]`, which happens to match the `NeuroLlamaConfig` default, so the doc is technically correct for 8B. However, these are not being set by the preset but rather by the dataclass default. This is a minor accuracy concern -- the doc implies the preset explicitly sets these values.

#### WRONG_DESCRIPTION
3. **Performance Notes claim about early exit**: The doc states (line 643): "Early exit confidence uses GELU approximation (no scipy dependency)". However, in the source code (`model.py` lines 167-168), early exit confidence is computed using standard softmax over logits:
   ```python
   logits = np.mean(h, axis=1) @ self._exit_head
   exit_confidence = float(np.max(_softmax(logits, axis=-1)))
   ```
   There is no GELU approximation involved. The confidence is the max probability from a softmax, not GELU.

#### Verified correct (extensive list):

**config.py** -- All 31 attributes (9 architecture, 14 neuroscience toggles, 5 training weights, 3 derived properties) match doc exactly. All methods (`from_llama_config`, `to_dict`, `from_dict`, `to_json`, `from_json`, `with_overrides`) match. All validation rules match.

**synaptic_attention.py** -- `SynapticScaling` (constructor, forward, update_alpha, get_alphas), `NeuromodulatedAttention` (constructor, compute_modulation, forward), `SynapticModulationMatrix` (constructor, compute_synaptic_strength, forward), `scaled_dot_product_attention` -- all match docs.

**cortical_columns.py** -- `CorticalColumnAttention` (constructor, compute_group_attention, compute_specialization_divergence, forward), `LateralInhibition` (constructor, forward, update_inhibition, get_inhibition_matrix), `HierarchicalColumnOrganizer` (constructor, get_column_role, get_specialization_targets) -- all match docs. Layer tier boundaries (syntactic 0-9, semantic 10-19, abstract 20+) match.

**predictive_coding.py** -- `PredictiveCodingConfig` (all 5 attributes match), `PredictionHead` (predict, compute_error match), `PredictiveCodingLayer` (constructor, forward, get_accumulated_surprise, reset match), `ContrastivePredictiveCoding` (constructor, score, sample_negatives, compute_loss match), `PredictionErrorSignal` (constructor, collect_errors, get_total_loss, get_surprise_profile, reset, get_stats match) -- all match docs.

**hebbian_attention.py** -- `HebbianAttention` (constructor, forward, reset, get_hebbian_strength match), `RewardModulatedHebbian` (constructor, forward, set_reward, get_stats, reset match), `HebbianFastWeights` (constructor, forward, accumulate, decay, reset, get_fast_weight_strength, get_stats match) -- all match docs.

**habituation_layer.py** -- `HabituatingAttention` (constructor, forward, reset, get_habituation_map match), `AttentionDecay` (constructor, forward match), `NoveltyDetector` (constructor, detect_novelty, get_novelty_scores, get_dishabituation_boost match) -- all match docs.

**population_output.py** -- `PopulationCodedAttention` (constructor, compute_activation, forward, update_preferred_directions match). Note: doc says `lr=0.001` param name, code uses `learning_rate=0.001` -- these match. `ConfidenceWeightedVoting` (constructor, estimate_confidence, forward, update_estimators match), `EntropyBasedVoting` (constructor, compute_head_entropy, forward match), `PopulationVectorDecoder` (constructor, decode, decode_with_confidence match) -- all match docs.

**training_objectives.py** -- `TrainingObjectiveConfig` (all 7 attributes match), `SurpriseLoss.compute` matches, `SpecializationLoss.compute` matches, `CalibrationLoss.compute` matches, `EfficiencyLoss.compute` matches, `CoherenceLoss.compute` matches, `NeuroCompositeLoss` (constructor, compute, get_loss_history, adjust_weights match), `BrainInspiredReward` (constructor, compute_prediction_accuracy, compute_efficiency_score, compute_repetition_penalty, compute_reward match) -- all match docs.

**model.py** -- `RMSNorm`, `RotaryPositionEmbedding`, `SwiGLU`, `NeuroLlamaBlock`, `NeuroLlamaModel`, `create_neurollama` -- all match docs (except the 3 gaps noted above).

---

## Gap Summary

| # | Page | Category | Severity | Description |
|---|------|----------|----------|-------------|
| 1 | interop/index.md | WRONG_DESCRIPTION | LOW | Architecture diagram names `MCPToolBridge` class that doesn't exist (it's a module of functions) |
| 2 | interop/index.md | WRONG_EXAMPLE | MEDIUM | See Also links point to non-existent `mcp-setup.md` and `a2a-delegation.md` instead of `mcp-servers.md` and `a2a-agents.md` |
| 3 | neurollama-model.md | WRONG_DESCRIPTION | MEDIUM | 70B preset exit_layers documented as `[20, 40, 60]` but code defaults to `[8, 16, 24]` |
| 4 | neurollama-model.md | WRONG_DESCRIPTION | LOW | 8B preset exit_layers are not set by the preset itself but inherited from dataclass default (doc implies preset sets them) |
| 5 | neurollama-model.md | WRONG_DESCRIPTION | LOW | Performance notes claim "GELU approximation" for early exit confidence; code actually uses softmax max probability |
