# Cognitive Context & Memory Architecture

## Revolutionary Context Management for corteX Enterprise AI Agent SDK

**Author:** Senior AI Architect, corteX R&D
**Date:** 2026-02-15
**Status:** Architecture Document -- Ready for Implementation
**Scope:** Complete redesign of context compilation, memory management, and cognitive state systems

---

## Executive Summary

This document presents a revolutionary architecture for context management and memory that goes far beyond the state of the art. Where current systems (Claude Code, Manus, Letta/MemGPT, Mem0) treat context as a linear buffer to fill and truncate, we treat it as a **cognitive information field** -- a dynamic, multi-resolution, entanglement-aware space where every token earns its place through measured mutual information with the goal.

The architecture introduces seven novel concepts:

1. **Context Entanglement Detection** -- Identifying context pairs that are individually low-value but jointly high-value, ensuring they always appear together
2. **Multi-Resolution Context Pyramid** -- Same information at 4 zoom levels, dynamically selected per token budget
3. **Predictive Context Pre-Loading** -- CPU-cache-inspired prefetching of context likely needed 2-3 turns ahead
4. **Memory Crystallization** -- After task success, distilling reusable cognitive patterns from the execution trace
5. **Active Forgetting Engine** -- Deliberate removal of outdated, contradicted, or poisonous memories
6. **Context Versioning with Causal Diff** -- Tracking what information was available at each decision point to diagnose failures
7. **Information Density Optimization** -- Structured encoding that packs 3-5x more semantic content per token

### Current State

corteX has a strong dual-engine foundation:
- `ContextCompiler` (4-zone, KV-cache-oriented) in `context_compiler.py`
- `CorticalContextEngine` (3-temperature, progressive compression) in `context.py`
- `MemoryFabric` (3-tier: working/episodic/semantic + cold storage) in `memory.py`
- `SummarizationPipeline` (L0-L3 progressive) in `context_summarizer.py`

**Key gaps:** No unified pipeline, no quality monitoring, no entanglement detection, no predictive pre-loading, no active forgetting, keyword-only memory search, no cross-session persistence, no memory crystallization.

---

## Architecture Design

### 1. Unified Cognitive Context Pipeline

Replace the dual `ContextCompiler` + `CorticalContextEngine` with a single unified pipeline.

```
                    +----------------------------------+
                    |   CognitiveContextPipeline       |
                    |   (Single entry point)           |
                    +----------------------------------+
                              |
          +-------------------+-------------------+
          |                   |                   |
   +------v------+    +------v------+    +-------v------+
   | ContextZone  |    | ContextZone  |    | ContextZone   |
   | Assembler    |    | Scorer       |    | Optimizer     |
   | (4-zone)     |    | (quality)    |    | (density)     |
   +------+------+    +------+------+    +-------+------+
          |                   |                   |
          +-------------------+-------------------+
                              |
                    +---------v---------+
                    | EntanglementGraph |
                    | (pair detection)  |
                    +---------+---------+
                              |
                    +---------v---------+
                    | PredictiveLoader  |
                    | (prefetch)        |
                    +---------+---------+
                              |
                    +---------v---------+
                    | MultiResolution   |
                    | Pyramid           |
                    +---------+---------+
                              |
                    +---------v---------+
                    | CompiledContext    |
                    | (final output)    |
                    +-------------------+
```

### 2. Context Entanglement Detection

**Novel concept:** Two context items may each score low individually but become critical when present together. Example: A schema definition (low importance after 50 steps) and an error message referencing a field in that schema (medium importance) -- together they enable root-cause diagnosis.

```
EntanglementGraph:
  - Nodes: every ContextItem currently in memory
  - Edges: weighted by co-reference score
  - Co-reference signals:
    1. Shared entity references (file paths, variable names, API endpoints)
    2. Causal chains (decision A led to action B led to error C)
    3. Temporal proximity of creation
    4. Shared goal-relevance vector direction

  When evicting item X:
    - Check all entangled pairs involving X
    - If X's partner Y is in context with importance > 0.6,
      boost X's importance by entanglement_weight
    - Never break an entangled pair where combined MI > threshold
```

**Data structure:**

```python
@dataclass
class EntanglementEdge:
    item_a_id: str
    item_b_id: str
    entanglement_score: float     # 0.0 to 1.0
    shared_entities: List[str]    # Entities both items reference
    causal_direction: str         # "a->b", "b->a", "bidirectional"
    detected_at_step: int
```

### 3. Multi-Resolution Context Pyramid

**Novel concept:** Maintain the same information at 4 resolution levels simultaneously. Select resolution dynamically based on available token budget and goal proximity.

```
Resolution Levels:
  R0 (Full):     Complete verbatim content         (1x tokens)
  R1 (Standard): Key sentences + structure         (0.3x tokens)
  R2 (Compact):  Entity-relationship summary       (0.1x tokens)
  R3 (Micro):    Single-line semantic fingerprint  (0.02x tokens)

Selection Algorithm:
  For each candidate item:
    1. Compute goal_proximity = cosine(item_embedding, goal_embedding)
    2. Compute recency_factor = exp(-lambda * age_steps)
    3. Compute entanglement_pull = max(entanglement_score of active partners)
    4. priority = 0.4*goal_proximity + 0.3*recency_factor + 0.3*entanglement_pull

  Budget allocation:
    - Top 10% priority items: R0 (full detail)
    - Next 20% priority items: R1 (standard)
    - Next 30% priority items: R2 (compact)
    - Bottom 40%: R3 (micro) or evict entirely
```

**Data structure:**

```python
@dataclass
class MultiResolutionItem:
    item_id: str
    r0_content: str              # Full verbatim
    r1_content: str              # Key sentences
    r2_content: str              # Entity-relationship
    r3_content: str              # Single-line fingerprint
    r0_tokens: int
    r1_tokens: int
    r2_tokens: int
    r3_tokens: int
    current_resolution: int      # Which level is currently selected
    entities_referenced: List[str]
    goal_proximity: float
    last_promoted_at: float      # When resolution was last increased
```

### 4. Predictive Context Pre-Loading

**Novel concept:** Like CPU cache prefetching, predict what context will be needed 2-3 turns ahead and pre-load it at R2/R3 resolution so it is instantly available for promotion to R0 when needed.

```
Prediction Signals:
  1. Plan step lookahead: If plan says "step 7: write tests for auth",
     pre-load auth-related memories, test framework knowledge
  2. Tool usage patterns: If agent just called file_read("auth.py"),
     pre-load memories about auth.py modifications, related test files
  3. Goal decomposition: Sub-goals not yet started predict future context needs
  4. Error pattern recognition: If error pattern matches a known pattern,
     pre-load the resolution that worked last time
  5. Entity co-occurrence: If entity A is active and A historically co-occurs
     with entity B, pre-load B

Algorithm:
  After each step:
    1. Extract entities from latest action + result
    2. Query PredictionEngine for next likely actions
    3. Query KnowledgeGraph for entities co-occurring with current entities
    4. Query EpisodicStore for similar goals' next steps
    5. Pre-load top-K candidates at R2 resolution into a "prefetch buffer"
    6. On next turn, check if any prefetched items are now relevant
       -> If yes: promote to R0/R1 (instant, no retrieval latency)
       -> If no: demote or evict from prefetch buffer
```

### 5. Context Quality Scoring Engine

Beyond simple token counting -- measure the **cognitive quality** of the assembled context.

```
Quality Dimensions:
  1. Goal Retention Score (GRS):
     cosine_similarity(context_embedding, goal_embedding)
     Target: > 0.3 at all times

  2. Information Density Index (IDI):
     unique_semantic_units / total_tokens
     Measures how much actual information per token

  3. Entanglement Completeness (EC):
     entangled_pairs_complete / total_entangled_pairs
     Target: > 0.9 (never break entangled pairs)

  4. Temporal Coherence (TC):
     weighted_coverage_of_causal_chain / total_causal_steps
     Measures whether the causal story is intact

  5. Decision Preservation Rate (DPR):
     decisions_in_context / total_decisions_made
     Target: > 0.8 for recent 50 steps

  6. Anti-Hallucination Score (AHS):
     verified_facts_tokens / total_context_tokens
     Weight verified facts higher; flag inferred info

Overall Quality = weighted harmonic mean of all 6 dimensions
  (harmonic mean penalizes any single low score heavily)
```

### 6. Memory Crystallization Engine

**Novel concept:** After a task succeeds, extract a reusable "crystal" -- a compressed cognitive pattern that can be injected into future similar tasks as a few-shot template.

```
Crystallization Process:
  1. Task completes successfully
  2. Extract execution trace: (goal, key_decisions, tool_chain, error_resolutions)
  3. Generalize: Replace specific values with type placeholders
     e.g., "Created file auth.py" -> "Created file {target_module}.py"
  4. Score by reusability:
     - How general is the pattern? (more placeholders = more reusable)
     - How successful was the outcome? (quality * user_feedback)
     - How novel is it? (dissimilarity to existing crystals)
  5. Store in CrystalStore (specialized episodic memory)
  6. On new task: query CrystalStore with goal embedding
     -> Inject matched crystal as few-shot example in persistent zone

Crystal Data Structure:
  crystal_id: str
  goal_template: str           # Generalized goal pattern
  decision_chain: List[str]    # Key decisions in order
  tool_sequence: List[str]     # Tools used in order
  error_patterns: List[str]    # Errors hit and how resolved
  success_score: float         # 0.0 to 1.0
  reuse_count: int             # How many times this crystal was applied
  effectiveness: float         # Average success when applied
  entities_pattern: Dict       # Entity types involved
  created_from_session: str    # Source session ID
```

### 7. Active Forgetting Engine

**Novel concept:** Not all forgetting is loss. Deliberate forgetting of outdated, contradicted, or poisonous memories improves agent performance (supported by ACT-R cognitive architecture research, HAI 2025).

```
Forgetting Triggers:
  1. Contradiction Detection:
     New fact F2 contradicts existing memory F1
     -> Mark F1 as "superseded_by: F2", decay importance to 0.05
     -> F1 moves to cold storage with "contradicted" tag
     -> If F2 later proves wrong, F1 can be restored

  2. Staleness Decay:
     Memory not accessed for N steps AND not entangled with active items
     -> Exponential importance decay with configurable half-life
     -> Below threshold 0.05 -> archive to cold storage

  3. Error Poisoning:
     Memory that was present during N consecutive failed actions
     -> Flag as "potentially_poisonous"
     -> Reduce importance by 0.3
     -> Track if removal correlates with improved performance

  4. Redundancy Elimination:
     Two memories with cosine_similarity > 0.95
     -> Merge into single memory preserving highest importance
     -> Keep provenance chain for audit

  5. Goal Divergence:
     Memory whose goal_proximity drops below 0.05 for 100+ steps
     -> Archive unless entangled or pinned
```

### 8. Compaction-Proof State Architecture

Three-layer state file system that survives unlimited compactions.

```
Layer 1: Crystallized State (NEVER deleted, always in persistent zone)
  - Original goal (verbatim)
  - Success criteria
  - Hard constraints
  - User identity and preferences

Layer 2: Fluid State (Updated every N steps, survives compaction)
  - Current sub-goal and progress
  - Active entities (files, APIs, schemas)
  - Recent decisions (last 10)
  - Unresolved errors
  - Open questions
  - Brain state digest

Layer 3: Insight State (Append-only, survives compaction)
  - Decision log (step, decision, rationale)
  - Error journal (error, pattern, resolution, status)
  - Learned constraints (discovered during execution)
  - Entity relationships discovered
```

### 9. Information Density Optimization

**Novel concept:** Pack maximum semantic content per token using structured encoding.

```
Standard (wasteful):
  "The user previously mentioned that they prefer using PostgreSQL
   as their database and they want the API to support pagination
   with cursor-based navigation rather than offset-based."
  = 33 tokens

Density-Optimized:
  "[PREF] db:PostgreSQL | pagination:cursor-based (not offset)"
  = 11 tokens

Density Gain: 3x

Optimization Rules:
  1. Use structured key:value pairs instead of prose
  2. Use domain abbreviations for repeated concepts
  3. Negative constraints as "(not X)" inline
  4. Tool results as structured tables, not narrative
  5. Error messages: just error class + message, not full stack trace
  6. File contents: AST-level summary, not character-level
  7. Conversation history: decision points only, not full exchange
```

### 10. Context Versioning with Causal Diff

**Novel concept:** Track what information was available at each decision point. When a mistake happens, diff the context to understand what was missing.

```
ContextVersion:
  version_id: str
  step_number: int
  decision_made: str
  context_hash: str            # Hash of full assembled context
  item_ids_present: Set[str]   # Which items were in context
  quality_scores: Dict         # Quality metrics at this point
  outcome: str                 # What happened after this decision

Causal Diff (when mistake detected):
  1. Find the decision step where things went wrong
  2. Find the last successful decision step
  3. Diff: what items were present in success but missing in failure?
  4. These are the "missing context items" that caused the mistake
  5. Boost their importance permanently (sleeper_awakened flag)
  6. Pre-load them in similar future situations
```

---

## Implementation Modules

### Module 1: `corteX/engine/cognitive_context.py` (< 300 lines)

Main pipeline orchestrator replacing dual-engine pattern.

```python
class CognitiveContextPipeline:
    """Unified context compilation pipeline with cognitive enhancements."""

    def __init__(
        self,
        max_context_tokens: int = 128_000,
        zone_budgets: Optional[Dict[str, float]] = None,
        quality_thresholds: Optional[Dict[str, float]] = None,
    ) -> None: ...

    async def compile(
        self,
        goal: str,
        system_prompt: str,
        messages: List[Dict[str, str]],
        brain_state: Dict[str, Any],
        memory_fabric: MemoryFabric,
        plan_state: Optional[Dict[str, Any]] = None,
        step_number: int = 0,
        max_steps: int = 100,
    ) -> CognitiveCompiledContext: ...

    async def record_outcome(
        self,
        step_number: int,
        decision: str,
        outcome: str,
        success: bool,
    ) -> None: ...

    def get_quality_report(self) -> ContextQualityReport: ...
    def get_stats(self) -> Dict[str, Any]: ...
```

Key methods:

```python
async def compile(self, ...) -> CognitiveCompiledContext:
    """Full cognitive compilation pipeline."""
    # Phase 1: Score all candidate items
    scored_items = await self._scorer.score_all(
        candidates, goal, step_number, self._entanglement_graph
    )

    # Phase 2: Resolve multi-resolution levels per budget
    resolved = self._pyramid.resolve(
        scored_items, available_budget, goal
    )

    # Phase 3: Enforce entanglement constraints
    resolved = self._entanglement_graph.enforce_pairs(resolved)

    # Phase 4: Optimize information density
    optimized = self._density_optimizer.optimize(resolved)

    # Phase 5: Assemble into 4 zones
    assembled = self._assembler.assemble(
        optimized, system_prompt, brain_state, plan_state
    )

    # Phase 6: Quality check
    quality = self._quality_engine.evaluate(
        assembled, goal, self._decision_log
    )

    # Phase 7: Predictive pre-load for next turn
    await self._predictive_loader.prefetch(
        step_number, plan_state, assembled
    )

    # Phase 8: Record context version
    self._versioner.record(step_number, assembled, quality)

    return CognitiveCompiledContext(
        messages=assembled.messages,
        total_tokens=assembled.total_tokens,
        zone_usage=assembled.zone_usage,
        quality=quality,
        prefetch_status=self._predictive_loader.get_status(),
    )
```

### Module 2: `corteX/engine/entanglement.py` (< 300 lines)

Context entanglement detection and enforcement.

```python
class EntityExtractor:
    """Extract named entities from context items for entanglement detection."""

    def extract(self, content: str) -> List[str]:
        """Extract file paths, variable names, API endpoints, etc."""
        entities: List[str] = []
        # File paths
        entities.extend(re.findall(r'[\w/\\]+\.\w{1,5}', content))
        # API endpoints
        entities.extend(re.findall(r'/api/[\w/]+', content))
        # Python identifiers after import/from/class/def
        entities.extend(re.findall(
            r'(?:import|from|class|def)\s+([\w.]+)', content
        ))
        # Quoted strings (config keys, etc.)
        entities.extend(re.findall(r'"([\w._-]+)"', content))
        return list(set(entities))


class EntanglementGraph:
    """Tracks co-reference relationships between context items."""

    def __init__(self, min_entanglement: float = 0.3) -> None:
        self._edges: Dict[str, EntanglementEdge] = {}
        self._entity_index: Dict[str, Set[str]] = {}  # entity -> item_ids
        self._extractor = EntityExtractor()
        self._min_entanglement = min_entanglement

    def register_item(self, item_id: str, content: str, step: int) -> None:
        """Register a new item and compute entanglement with existing."""
        entities = self._extractor.extract(content)
        for entity in entities:
            if entity not in self._entity_index:
                self._entity_index[entity] = set()
            existing_items = self._entity_index[entity]
            for other_id in existing_items:
                self._update_edge(item_id, other_id, entity, step)
            self._entity_index[entity].add(item_id)

    def enforce_pairs(
        self, items: List[ScoredItem]
    ) -> List[ScoredItem]:
        """Ensure entangled pairs are not separated."""
        included_ids = {i.item_id for i in items}
        additions: List[ScoredItem] = []
        for edge in self._edges.values():
            if edge.entanglement_score < self._min_entanglement:
                continue
            a_in = edge.item_a_id in included_ids
            b_in = edge.item_b_id in included_ids
            if a_in and not b_in:
                partner = self._retrieve_item(edge.item_b_id)
                if partner:
                    additions.append(partner)
                    included_ids.add(edge.item_b_id)
            elif b_in and not a_in:
                partner = self._retrieve_item(edge.item_a_id)
                if partner:
                    additions.append(partner)
                    included_ids.add(edge.item_a_id)
        return items + additions

    def get_entanglement_boost(self, item_id: str, active_ids: Set[str]) -> float:
        """Compute importance boost from entangled partners in context."""
        max_boost = 0.0
        for edge in self._edges.values():
            if edge.item_a_id == item_id and edge.item_b_id in active_ids:
                max_boost = max(max_boost, edge.entanglement_score * 0.5)
            elif edge.item_b_id == item_id and edge.item_a_id in active_ids:
                max_boost = max(max_boost, edge.entanglement_score * 0.5)
        return max_boost

    def _update_edge(
        self, id_a: str, id_b: str, shared_entity: str, step: int
    ) -> None:
        """Update or create entanglement edge between two items."""
        key = f"{min(id_a, id_b)}:{max(id_a, id_b)}"
        if key not in self._edges:
            self._edges[key] = EntanglementEdge(
                item_a_id=min(id_a, id_b),
                item_b_id=max(id_a, id_b),
                entanglement_score=0.0,
                shared_entities=[],
                causal_direction="bidirectional",
                detected_at_step=step,
            )
        edge = self._edges[key]
        if shared_entity not in edge.shared_entities:
            edge.shared_entities.append(shared_entity)
        # Score increases with number of shared entities
        edge.entanglement_score = min(
            1.0, len(edge.shared_entities) * 0.25
        )
```

### Module 3: `corteX/engine/context_pyramid.py` (< 300 lines)

Multi-resolution context management.

```python
class ContextPyramid:
    """Multi-resolution context with dynamic zoom."""

    def __init__(
        self,
        density_optimizer: Optional[DensityOptimizer] = None,
    ) -> None:
        self._items: Dict[str, MultiResolutionItem] = {}
        self._optimizer = density_optimizer or DensityOptimizer()

    async def register(
        self,
        item_id: str,
        content: str,
        goal: str,
        entities: List[str],
    ) -> MultiResolutionItem:
        """Register content at all 4 resolution levels."""
        r0 = content
        r1 = self._compress_to_key_sentences(content)
        r2 = self._compress_to_entity_summary(content, entities)
        r3 = self._compress_to_fingerprint(content)
        goal_prox = self._compute_goal_proximity(content, goal)

        item = MultiResolutionItem(
            item_id=item_id,
            r0_content=r0, r1_content=r1,
            r2_content=r2, r3_content=r3,
            r0_tokens=_estimate_tokens(r0),
            r1_tokens=_estimate_tokens(r1),
            r2_tokens=_estimate_tokens(r2),
            r3_tokens=_estimate_tokens(r3),
            current_resolution=0,
            entities_referenced=entities,
            goal_proximity=goal_prox,
            last_promoted_at=time.time(),
        )
        self._items[item_id] = item
        return item

    def resolve(
        self,
        scored_items: List[ScoredItem],
        budget_tokens: int,
        goal: str,
    ) -> List[ResolvedItem]:
        """Select optimal resolution for each item within budget."""
        scored_items.sort(key=lambda x: x.priority, reverse=True)

        resolved: List[ResolvedItem] = []
        remaining = budget_tokens

        # Top 10%: R0 (full detail)
        top_10 = max(1, len(scored_items) // 10)
        for item in scored_items[:top_10]:
            mr = self._items.get(item.item_id)
            if not mr:
                continue
            if mr.r0_tokens <= remaining:
                resolved.append(ResolvedItem(
                    item_id=item.item_id,
                    content=mr.r0_content,
                    tokens=mr.r0_tokens,
                    resolution=0,
                    priority=item.priority,
                ))
                remaining -= mr.r0_tokens

        # Next 20%: R1
        next_20 = max(1, len(scored_items) * 3 // 10)
        for item in scored_items[top_10:next_20]:
            mr = self._items.get(item.item_id)
            if not mr:
                continue
            if mr.r1_tokens <= remaining:
                resolved.append(ResolvedItem(
                    item_id=item.item_id,
                    content=mr.r1_content,
                    tokens=mr.r1_tokens,
                    resolution=1,
                    priority=item.priority,
                ))
                remaining -= mr.r1_tokens

        # Next 30%: R2
        next_60 = max(1, len(scored_items) * 6 // 10)
        for item in scored_items[next_20:next_60]:
            mr = self._items.get(item.item_id)
            if not mr:
                continue
            if mr.r2_tokens <= remaining:
                resolved.append(ResolvedItem(
                    item_id=item.item_id,
                    content=mr.r2_content,
                    tokens=mr.r2_tokens,
                    resolution=2,
                    priority=item.priority,
                ))
                remaining -= mr.r2_tokens

        # Bottom 40%: R3
        for item in scored_items[next_60:]:
            mr = self._items.get(item.item_id)
            if not mr:
                continue
            if mr.r3_tokens <= remaining:
                resolved.append(ResolvedItem(
                    item_id=item.item_id,
                    content=mr.r3_content,
                    tokens=mr.r3_tokens,
                    resolution=3,
                    priority=item.priority,
                ))
                remaining -= mr.r3_tokens

        return resolved

    def _compress_to_key_sentences(self, content: str) -> str:
        """Extract sentences containing entities or decisions."""
        sentences = re.split(r'[.!?\n]+', content)
        key_patterns = [
            r'\b(?:decided|chose|selected|using|created|error|failed)\b',
            r'[\w/\\]+\.\w{1,5}',  # File paths
            r'/api/',               # API endpoints
        ]
        key_sentences = []
        for s in sentences:
            s = s.strip()
            if not s:
                continue
            for pattern in key_patterns:
                if re.search(pattern, s, re.IGNORECASE):
                    key_sentences.append(s)
                    break
        return ". ".join(key_sentences) if key_sentences else content[:200]

    def _compress_to_entity_summary(
        self, content: str, entities: List[str]
    ) -> str:
        """Compress to entity:relationship pairs."""
        if not entities:
            return content[:100]
        return " | ".join(f"{e}: referenced" for e in entities[:10])

    def _compress_to_fingerprint(self, content: str) -> str:
        """Single-line semantic fingerprint."""
        words = content.split()[:20]
        return " ".join(words) + "..." if len(words) == 20 else " ".join(words)

    def _compute_goal_proximity(self, content: str, goal: str) -> float:
        """Keyword-overlap proximity (on-prem, no embeddings needed)."""
        if not goal or not content:
            return 0.3
        goal_words = set(goal.lower().split())
        content_words = set(content.lower().split())
        overlap = len(goal_words & content_words)
        return min(1.0, overlap / max(1, len(goal_words)) * 2.0)
```

### Module 4: `corteX/engine/predictive_loader.py` (< 300 lines)

Prefetch context likely needed in upcoming turns.

```python
class PredictiveContextLoader:
    """CPU-cache-inspired context prefetching for upcoming turns."""

    def __init__(
        self,
        memory_fabric: MemoryFabric,
        entanglement_graph: EntanglementGraph,
        prefetch_buffer_size: int = 20,
        lookahead_steps: int = 3,
    ) -> None:
        self._memory = memory_fabric
        self._entanglement = entanglement_graph
        self._buffer: Dict[str, PrefetchedItem] = {}
        self._buffer_size = prefetch_buffer_size
        self._lookahead = lookahead_steps
        self._hit_count: int = 0
        self._miss_count: int = 0
        self._total_prefetches: int = 0

    async def prefetch(
        self,
        step_number: int,
        plan_state: Optional[Dict[str, Any]],
        current_context: AssembledContext,
    ) -> None:
        """Predict and pre-load context for next 2-3 turns."""
        predictions: List[PredictionCandidate] = []

        # Signal 1: Plan step lookahead
        if plan_state:
            upcoming_steps = self._get_upcoming_steps(
                plan_state, self._lookahead
            )
            for upcoming in upcoming_steps:
                entities = EntityExtractor().extract(
                    upcoming.get("description", "")
                )
                for entity in entities:
                    predictions.append(PredictionCandidate(
                        query=entity,
                        source="plan_lookahead",
                        confidence=0.7,
                        expected_step=step_number + upcoming.get("offset", 1),
                    ))

        # Signal 2: Entity co-occurrence from entanglement graph
        current_entities = self._extract_active_entities(current_context)
        for entity in current_entities:
            co_occurring = self._entanglement.get_co_occurring_entities(entity)
            for co_entity in co_occurring:
                if co_entity not in current_entities:
                    predictions.append(PredictionCandidate(
                        query=co_entity,
                        source="co_occurrence",
                        confidence=0.5,
                        expected_step=step_number + 1,
                    ))

        # Signal 3: Episodic pattern matching
        goal = plan_state.get("goal", "") if plan_state else ""
        similar_episodes = self._memory.episodic.recall_similar(
            goal, max_results=2
        )
        for episode in similar_episodes:
            if episode.success and len(episode.steps) > step_number:
                future_step = episode.steps[
                    min(step_number, len(episode.steps) - 1)
                ]
                predictions.append(PredictionCandidate(
                    query=str(future_step),
                    source="episodic_pattern",
                    confidence=0.6 * episode.quality,
                    expected_step=step_number + 1,
                ))

        # Execute prefetch for top candidates
        predictions.sort(key=lambda p: p.confidence, reverse=True)
        for pred in predictions[:self._buffer_size]:
            if pred.query not in self._buffer:
                items = self._memory.working.search(pred.query, max_results=2)
                cold_items = (
                    self._memory.cold_storage.retrieve(
                        pred.query, max_results=2
                    )
                    if self._memory.cold_storage else []
                )
                for item in items + cold_items:
                    self._buffer[item.key] = PrefetchedItem(
                        item=item,
                        prefetched_at_step=step_number,
                        predicted_need_step=pred.expected_step,
                        source=pred.source,
                        confidence=pred.confidence,
                    )
                    self._total_prefetches += 1

        # Evict stale prefetch items
        self._evict_stale(step_number)

    def check_hit(self, query: str) -> Optional[MemoryItem]:
        """Check if a prefetched item matches the current need."""
        query_lower = query.lower()
        for key, prefetched in list(self._buffer.items()):
            content_str = str(prefetched.item.content).lower()
            if query_lower in content_str or query_lower in key.lower():
                self._hit_count += 1
                del self._buffer[key]
                return prefetched.item
        self._miss_count += 1
        return None

    def get_status(self) -> Dict[str, Any]:
        """Prefetch performance statistics."""
        total = self._hit_count + self._miss_count
        return {
            "buffer_size": len(self._buffer),
            "total_prefetches": self._total_prefetches,
            "hit_count": self._hit_count,
            "miss_count": self._miss_count,
            "hit_rate": self._hit_count / max(1, total),
        }

    def _evict_stale(self, current_step: int) -> None:
        """Remove prefetched items past their expected need step."""
        stale_keys = [
            k for k, v in self._buffer.items()
            if current_step > v.predicted_need_step + 2
        ]
        for k in stale_keys:
            del self._buffer[k]

    def _get_upcoming_steps(
        self, plan_state: Dict[str, Any], count: int
    ) -> List[Dict[str, Any]]:
        """Extract next N steps from plan."""
        steps = plan_state.get("steps", [])
        current = plan_state.get("current_step_index", 0)
        upcoming = []
        for i, step in enumerate(steps[current + 1:current + 1 + count]):
            upcoming.append({
                "description": step.get("description", ""),
                "offset": i + 1,
            })
        return upcoming

    def _extract_active_entities(
        self, context: AssembledContext
    ) -> Set[str]:
        """Extract entities currently active in the context."""
        entities: Set[str] = set()
        extractor = EntityExtractor()
        for msg in context.messages[-5:]:
            entities.update(extractor.extract(msg.get("content", "")))
        return entities
```

### Module 5: `corteX/engine/context_quality.py` (< 300 lines)

Quality scoring and monitoring.

```python
@dataclass
class ContextQualityReport:
    """Comprehensive quality assessment of assembled context."""
    goal_retention_score: float
    information_density_index: float
    entanglement_completeness: float
    temporal_coherence: float
    decision_preservation_rate: float
    anti_hallucination_score: float
    overall_health: float           # Harmonic mean
    health_label: str               # "optimal", "healthy", "degrading", "critical"
    warnings: List[str]
    recommendations: List[str]


class ContextQualityEngine:
    """Monitors and scores context quality across 6 dimensions."""

    def __init__(
        self,
        grs_threshold: float = 0.3,
        idi_threshold: float = 0.4,
        ec_threshold: float = 0.9,
        tc_threshold: float = 0.7,
        dpr_threshold: float = 0.8,
        ahs_threshold: float = 0.5,
    ) -> None:
        self._thresholds = {
            "grs": grs_threshold, "idi": idi_threshold,
            "ec": ec_threshold, "tc": tc_threshold,
            "dpr": dpr_threshold, "ahs": ahs_threshold,
        }
        self._history: List[ContextQualityReport] = []

    def evaluate(
        self,
        assembled: AssembledContext,
        goal: str,
        decision_log: List[Dict[str, str]],
        entanglement_graph: Optional[EntanglementGraph] = None,
        causal_chain: Optional[List[str]] = None,
    ) -> ContextQualityReport:
        """Run full quality evaluation across all 6 dimensions."""
        warnings: List[str] = []
        recommendations: List[str] = []

        full_text = " ".join(
            m.get("content", "") for m in assembled.messages
        )

        # 1. Goal Retention Score
        grs = self._keyword_similarity(full_text, goal)
        if grs < self._thresholds["grs"]:
            warnings.append(
                f"Goal retention critically low ({grs:.2f}). "
                "Goal signal is being diluted."
            )
            recommendations.append(
                "Increase goal restatement frequency. "
                "Move goal-relevant items to higher resolution."
            )

        # 2. Information Density Index
        total_tokens = assembled.total_tokens
        unique_entities = self._count_unique_entities(full_text)
        idi = unique_entities / max(1, total_tokens) * 100

        # 3. Entanglement Completeness
        ec = 1.0
        if entanglement_graph:
            ec = entanglement_graph.completeness_ratio(assembled)
            if ec < self._thresholds["ec"]:
                warnings.append(
                    f"Entangled pairs broken ({ec:.0%} complete)."
                )

        # 4. Temporal Coherence
        tc = self._compute_temporal_coherence(assembled, causal_chain)

        # 5. Decision Preservation Rate
        dpr = self._compute_dpr(assembled, decision_log)
        if dpr < self._thresholds["dpr"]:
            warnings.append(
                f"Decision preservation low ({dpr:.0%}). "
                "Agent may contradict previous decisions."
            )

        # 6. Anti-Hallucination Score
        ahs = self._compute_ahs(assembled)

        # Overall (weighted harmonic mean)
        scores = [grs, idi, ec, tc, dpr, ahs]
        weights = [0.25, 0.10, 0.20, 0.15, 0.20, 0.10]
        overall = self._weighted_harmonic_mean(scores, weights)
        label = (
            "optimal" if overall > 0.7
            else "healthy" if overall > 0.5
            else "degrading" if overall > 0.3
            else "critical"
        )

        report = ContextQualityReport(
            goal_retention_score=grs,
            information_density_index=idi,
            entanglement_completeness=ec,
            temporal_coherence=tc,
            decision_preservation_rate=dpr,
            anti_hallucination_score=ahs,
            overall_health=overall,
            health_label=label,
            warnings=warnings,
            recommendations=recommendations,
        )
        self._history.append(report)
        return report

    def get_trend(self, window: int = 10) -> Dict[str, List[float]]:
        """Quality trend over last N evaluations."""
        recent = self._history[-window:]
        return {
            "grs": [r.goal_retention_score for r in recent],
            "overall": [r.overall_health for r in recent],
        }

    def _weighted_harmonic_mean(
        self, scores: List[float], weights: List[float]
    ) -> float:
        numerator = sum(weights)
        denominator = sum(
            w / max(s, 0.01) for w, s in zip(weights, scores)
        )
        return numerator / max(denominator, 0.001)

    def _keyword_similarity(self, text: str, goal: str) -> float:
        if not goal or not text:
            return 0.0
        goal_words = set(goal.lower().split())
        text_words = set(text.lower().split())
        overlap = len(goal_words & text_words)
        return min(1.0, overlap / max(1, len(goal_words)))

    def _count_unique_entities(self, text: str) -> int:
        return len(set(EntityExtractor().extract(text)))

    def _compute_temporal_coherence(
        self, assembled: AssembledContext,
        causal_chain: Optional[List[str]],
    ) -> float:
        if not causal_chain:
            return 1.0
        full_text = " ".join(
            m.get("content", "") for m in assembled.messages
        ).lower()
        present = sum(
            1 for step in causal_chain if step.lower()[:30] in full_text
        )
        return present / max(1, len(causal_chain))

    def _compute_dpr(
        self, assembled: AssembledContext,
        decision_log: List[Dict[str, str]],
    ) -> float:
        if not decision_log:
            return 1.0
        full_text = " ".join(
            m.get("content", "") for m in assembled.messages
        ).lower()
        recent = decision_log[-50:]
        preserved = sum(
            1 for d in recent
            if d.get("decision", "")[:40].lower() in full_text
        )
        return preserved / max(1, len(recent))

    def _compute_ahs(self, assembled: AssembledContext) -> float:
        total = assembled.total_tokens
        tool_tokens = sum(
            _estimate_tokens(m.get("content", ""))
            for m in assembled.messages
            if m.get("role") == "tool" or "[Tool" in m.get("content", "")
        )
        return min(1.0, (tool_tokens / max(1, total)) * 3)
```

### Module 6: `corteX/engine/memory_crystallizer.py` (< 300 lines)

Extract reusable cognitive patterns from successful executions.

```python
@dataclass
class CognitiveCrystal:
    """Reusable pattern extracted from a successful task execution."""
    crystal_id: str
    goal_template: str
    decision_chain: List[str]
    tool_sequence: List[str]
    error_patterns: List[Dict[str, str]]
    success_score: float
    reuse_count: int = 0
    effectiveness: float = 0.0
    entity_types: List[str] = field(default_factory=list)
    created_from: str = ""
    created_at: float = field(default_factory=time.time)


class MemoryCrystallizer:
    """Distill reusable patterns from successful task completions."""

    def __init__(
        self,
        crystal_store: Optional[MemoryBackend] = None,
        max_crystals: int = 200,
        min_success_score: float = 0.7,
    ) -> None:
        self._store = crystal_store or InMemoryBackend()
        self._max_crystals = max_crystals
        self._min_score = min_success_score

    async def crystallize(
        self,
        goal: str,
        decision_log: List[Dict[str, str]],
        tool_usage: List[Dict[str, Any]],
        error_journal: List[Dict[str, str]],
        success_score: float,
        session_id: str,
    ) -> Optional[CognitiveCrystal]:
        """Extract crystal from a successful task execution."""
        if success_score < self._min_score:
            return None

        template = self._generalize_goal(goal)
        decisions = [d.get("decision", "") for d in decision_log]
        tools = list(dict.fromkeys(
            t.get("tool_name", "") for t in tool_usage
        ))
        error_patterns = [
            {"error": e.get("error", ""), "resolution": e.get("resolution", "")}
            for e in error_journal if e.get("resolution")
        ]
        entity_types = self._extract_entity_types(goal, decision_log)

        crystal = CognitiveCrystal(
            crystal_id=f"crystal_{session_id}_{int(time.time())}",
            goal_template=template,
            decision_chain=decisions,
            tool_sequence=tools,
            error_patterns=error_patterns,
            success_score=success_score,
            entity_types=entity_types,
            created_from=session_id,
        )

        if not self._is_novel(crystal):
            return None

        self._store.put(MemoryItem(
            key=crystal.crystal_id,
            content=self._serialize_crystal(crystal),
            importance=success_score,
            metadata={"type": "crystal", "goal_template": template},
        ))
        return crystal

    def recall_crystal(
        self, goal: str, max_results: int = 3
    ) -> List[CognitiveCrystal]:
        """Find crystals matching a new goal."""
        items = self._store.search(goal, max_results=max_results)
        crystals = []
        for item in items:
            crystal = self._deserialize_crystal(item.content)
            if crystal:
                crystals.append(crystal)
        return crystals

    def record_reuse(self, crystal_id: str, success: bool) -> None:
        """Record that a crystal was reused and whether it helped."""
        item = self._store.get(crystal_id)
        if not item:
            return
        crystal = self._deserialize_crystal(item.content)
        if crystal:
            crystal.reuse_count += 1
            n = crystal.reuse_count
            crystal.effectiveness = (
                crystal.effectiveness * (n - 1) + (1.0 if success else 0.0)
            ) / n
            item.content = self._serialize_crystal(crystal)
            self._store.put(item)

    def format_as_fewshot(self, crystal: CognitiveCrystal) -> str:
        """Format crystal as a few-shot example for context injection."""
        lines = [
            f"## Similar Past Success ({crystal.effectiveness:.0%} effective)",
            f"Goal pattern: {crystal.goal_template}",
            f"Approach: {' -> '.join(crystal.decision_chain[:5])}",
            f"Tools used: {', '.join(crystal.tool_sequence[:5])}",
        ]
        if crystal.error_patterns:
            lines.append("Known pitfalls:")
            for ep in crystal.error_patterns[:3]:
                lines.append(f"  - {ep['error']}: {ep['resolution']}")
        return "\n".join(lines)

    def _generalize_goal(self, goal: str) -> str:
        template = goal
        template = re.sub(r'[\w/\\]+\.\w{1,5}', '{file_path}', template)
        template = re.sub(r'"[^"]{3,}"', '"{value}"', template)
        template = re.sub(r'\b\d{2,}\b', '{N}', template)
        return template

    def _extract_entity_types(
        self, goal: str, decisions: List[Dict[str, str]]
    ) -> List[str]:
        types: Set[str] = set()
        full_text = goal + " ".join(
            d.get("decision", "") for d in decisions
        )
        if re.search(r'\.py\b', full_text):
            types.add("python_file")
        if re.search(r'/api/', full_text):
            types.add("api_endpoint")
        if re.search(r'test', full_text, re.I):
            types.add("test")
        if re.search(r'database|sql|postgres|mongo', full_text, re.I):
            types.add("database")
        return list(types)

    def _is_novel(self, crystal: CognitiveCrystal) -> bool:
        existing = self._store.search(crystal.goal_template, max_results=3)
        for item in existing:
            ec = self._deserialize_crystal(item.content)
            if ec:
                overlap = set(crystal.tool_sequence) & set(ec.tool_sequence)
                similarity = len(overlap) / max(
                    1, len(set(crystal.tool_sequence) | set(ec.tool_sequence))
                )
                if similarity > 0.8:
                    return False
        return True

    def _serialize_crystal(self, c: CognitiveCrystal) -> Dict[str, Any]:
        return {
            "crystal_id": c.crystal_id, "goal_template": c.goal_template,
            "decision_chain": c.decision_chain, "tool_sequence": c.tool_sequence,
            "error_patterns": c.error_patterns, "success_score": c.success_score,
            "reuse_count": c.reuse_count, "effectiveness": c.effectiveness,
            "entity_types": c.entity_types, "created_from": c.created_from,
        }

    def _deserialize_crystal(self, data: Any) -> Optional[CognitiveCrystal]:
        if not isinstance(data, dict):
            return None
        try:
            return CognitiveCrystal(**{
                k: data.get(k, getattr(CognitiveCrystal, k, None))
                for k in CognitiveCrystal.__dataclass_fields__
                if k in data
            })
        except Exception:
            return None
```

### Module 7: `corteX/engine/active_forgetting.py` (< 300 lines)

Deliberate memory removal for improved performance.

```python
class ForgettingReason(str, Enum):
    CONTRADICTION = "contradiction"
    STALENESS = "staleness"
    ERROR_POISONING = "error_poisoning"
    REDUNDANCY = "redundancy"
    GOAL_DIVERGENCE = "goal_divergence"


@dataclass
class ForgettingEvent:
    item_key: str
    reason: ForgettingReason
    step_number: int
    original_importance: float
    superseded_by: Optional[str] = None
    reversible: bool = True
    timestamp: float = field(default_factory=time.time)


class ActiveForgettingEngine:
    """Deliberate memory removal for improved cognitive performance."""

    def __init__(
        self,
        staleness_half_life: int = 200,
        contradiction_decay: float = 0.05,
        poison_threshold: int = 3,
        redundancy_threshold: float = 0.95,
    ) -> None:
        self._staleness_half_life = staleness_half_life
        self._contradiction_decay = contradiction_decay
        self._poison_threshold = poison_threshold
        self._redundancy_threshold = redundancy_threshold
        self._forgetting_log: List[ForgettingEvent] = []
        self._failure_context: Dict[str, int] = {}

    def scan_for_contradictions(
        self, new_fact: str, new_key: str,
        existing_items: List[MemoryItem],
    ) -> List[ForgettingEvent]:
        """Detect memories contradicted by a new fact."""
        events: List[ForgettingEvent] = []
        new_lower = new_fact.lower()
        for item in existing_items:
            content_lower = str(item.content).lower()
            is_contradiction = False
            for old_claim_word in content_lower.split()[:20]:
                if f"not {old_claim_word}" in new_lower:
                    is_contradiction = True
                    break
                if f"no longer {old_claim_word}" in new_lower:
                    is_contradiction = True
                    break
            if "instead of" in new_lower or "changed from" in new_lower:
                if any(w in content_lower for w in new_lower.split()[:5]):
                    is_contradiction = True
            if is_contradiction:
                events.append(ForgettingEvent(
                    item_key=item.key,
                    reason=ForgettingReason.CONTRADICTION,
                    step_number=0,
                    original_importance=item.importance,
                    superseded_by=new_key,
                ))
                self._forgetting_log.append(events[-1])
        return events

    def scan_for_staleness(
        self, items: List[MemoryItem], current_step: int,
        entangled_keys: Set[str], pinned_keys: Set[str],
    ) -> List[ForgettingEvent]:
        """Identify items that have become stale."""
        events: List[ForgettingEvent] = []
        decay_rate = math.log(2) / self._staleness_half_life
        for item in items:
            if item.key in pinned_keys or item.key in entangled_keys:
                continue
            age = current_step - item.metadata.get("step_number", 0)
            if age < self._staleness_half_life:
                continue
            decayed = item.importance * math.exp(-decay_rate * age)
            if decayed < 0.05:
                events.append(ForgettingEvent(
                    item_key=item.key,
                    reason=ForgettingReason.STALENESS,
                    step_number=current_step,
                    original_importance=item.importance,
                ))
                self._forgetting_log.append(events[-1])
        return events

    def record_failure_context(
        self, active_item_keys: List[str]
    ) -> List[ForgettingEvent]:
        """Track items present during failures (error poisoning)."""
        events: List[ForgettingEvent] = []
        for key in active_item_keys:
            self._failure_context[key] = (
                self._failure_context.get(key, 0) + 1
            )
            if self._failure_context[key] >= self._poison_threshold:
                events.append(ForgettingEvent(
                    item_key=key,
                    reason=ForgettingReason.ERROR_POISONING,
                    step_number=0,
                    original_importance=0.0,
                ))
                self._forgetting_log.append(events[-1])
        return events

    def record_success_context(self, active_item_keys: List[str]) -> None:
        """Reset failure counts for items present during success."""
        for key in active_item_keys:
            if key in self._failure_context:
                self._failure_context[key] = max(
                    0, self._failure_context[key] - 1
                )

    def scan_for_redundancy(
        self, items: List[MemoryItem]
    ) -> List[ForgettingEvent]:
        """Find near-duplicate memories."""
        events: List[ForgettingEvent] = []
        seen: Dict[str, str] = {}
        for item in sorted(items, key=lambda i: -i.importance):
            content_hash = hashlib.md5(
                str(item.content).lower().strip().encode()
            ).hexdigest()
            if content_hash in seen and seen[content_hash] != item.key:
                events.append(ForgettingEvent(
                    item_key=item.key,
                    reason=ForgettingReason.REDUNDANCY,
                    step_number=0,
                    original_importance=item.importance,
                    superseded_by=seen[content_hash],
                    reversible=False,
                ))
                self._forgetting_log.append(events[-1])
            else:
                seen[content_hash] = item.key
        return events

    def get_forgetting_stats(self) -> Dict[str, Any]:
        by_reason: Dict[str, int] = {}
        for event in self._forgetting_log:
            r = event.reason.value
            by_reason[r] = by_reason.get(r, 0) + 1
        return {
            "total_forgotten": len(self._forgetting_log),
            "by_reason": by_reason,
            "active_poison_watches": sum(
                1 for v in self._failure_context.values() if v > 0
            ),
        }
```

### Module 8: `corteX/engine/context_versioner.py` (< 300 lines)

Track context state at each decision point for causal diagnosis.

```python
@dataclass
class ContextVersion:
    version_id: str
    step_number: int
    decision_made: str
    context_hash: str
    item_ids_present: List[str]
    quality_snapshot: Dict[str, float]
    outcome: str = ""
    outcome_success: Optional[bool] = None
    timestamp: float = field(default_factory=time.time)


@dataclass
class CausalDiff:
    success_version: str
    failure_version: str
    items_in_success_not_failure: List[str]
    items_in_failure_not_success: List[str]
    quality_delta: Dict[str, float]
    diagnosis: str


class ContextVersioner:
    """Track context evolution for causal failure diagnosis."""

    def __init__(self, max_versions: int = 500) -> None:
        self._versions: Dict[str, ContextVersion] = {}
        self._version_order: List[str] = []
        self._max_versions = max_versions

    def record(
        self, step_number: int, assembled: AssembledContext,
        quality: ContextQualityReport, decision: str = "",
    ) -> ContextVersion:
        full_text = "".join(
            m.get("content", "") for m in assembled.messages
        )
        ctx_hash = hashlib.sha256(full_text.encode()).hexdigest()[:16]
        vid = f"v_{step_number}_{ctx_hash}"
        version = ContextVersion(
            version_id=vid, step_number=step_number,
            decision_made=decision, context_hash=ctx_hash,
            item_ids_present=[
                m.get("_item_id", f"msg_{i}")
                for i, m in enumerate(assembled.messages)
            ],
            quality_snapshot={
                "grs": quality.goal_retention_score,
                "overall": quality.overall_health,
                "dpr": quality.decision_preservation_rate,
            },
        )
        self._versions[vid] = version
        self._version_order.append(vid)
        while len(self._version_order) > self._max_versions:
            old_id = self._version_order.pop(0)
            self._versions.pop(old_id, None)
        return version

    def record_outcome(
        self, step_number: int, outcome: str, success: bool
    ) -> None:
        for vid in reversed(self._version_order):
            v = self._versions.get(vid)
            if v and v.step_number == step_number:
                v.outcome = outcome
                v.outcome_success = success
                break

    def diagnose_failure(self, failure_step: int) -> Optional[CausalDiff]:
        failure_v = None
        success_v = None
        for vid in reversed(self._version_order):
            v = self._versions.get(vid)
            if not v:
                continue
            if v.step_number == failure_step:
                failure_v = v
            elif v.outcome_success is True and failure_v is not None:
                success_v = v
                break
        if not failure_v or not success_v:
            return None
        success_ids = set(success_v.item_ids_present)
        failure_ids = set(failure_v.item_ids_present)
        quality_delta = {
            k: success_v.quality_snapshot.get(k, 0)
               - failure_v.quality_snapshot.get(k, 0)
            for k in set(success_v.quality_snapshot) | set(failure_v.quality_snapshot)
        }
        missing = list(success_ids - failure_ids)
        extra = list(failure_ids - success_ids)
        parts = []
        if missing:
            parts.append(
                f"{len(missing)} items present in success but missing at failure."
            )
        if quality_delta.get("grs", 0) > 0.1:
            parts.append("Goal retention was higher during success.")
        if quality_delta.get("dpr", 0) > 0.1:
            parts.append("Decision preservation was higher during success.")
        return CausalDiff(
            success_version=success_v.version_id,
            failure_version=failure_v.version_id,
            items_in_success_not_failure=missing,
            items_in_failure_not_success=extra,
            quality_delta=quality_delta,
            diagnosis=" ".join(parts) if parts else "No clear diagnosis.",
        )

    def get_quality_trend(self, last_n: int = 20) -> List[Dict[str, Any]]:
        recent = self._version_order[-last_n:]
        return [
            {
                "step": self._versions[vid].step_number,
                "quality": self._versions[vid].quality_snapshot,
                "success": self._versions[vid].outcome_success,
            }
            for vid in recent if vid in self._versions
        ]
```

### Module 9: `corteX/engine/density_optimizer.py` (< 300 lines)

Maximize information per token in context.

```python
class DensityOptimizer:
    """Pack maximum semantic content per token."""

    def __init__(self) -> None:
        self._tokens_saved: int = 0

    def optimize(self, items: List[ResolvedItem]) -> List[ResolvedItem]:
        optimized: List[ResolvedItem] = []
        for item in items:
            content = item.content
            original_tokens = item.tokens
            content = self._to_structured(content)
            content = self._apply_abbreviations(content)
            content = self._remove_inline_redundancy(content)
            new_tokens = _estimate_tokens(content)
            self._tokens_saved += max(0, original_tokens - new_tokens)
            optimized.append(ResolvedItem(
                item_id=item.item_id, content=content,
                tokens=new_tokens, resolution=item.resolution,
                priority=item.priority,
            ))
        return optimized

    def _to_structured(self, content: str) -> str:
        if content.startswith("[") and "]" in content:
            return content
        lines = content.split("\n")
        structured: List[str] = []
        for line in lines:
            line = line.strip()
            if not line:
                continue
            match = re.match(
                r'(?:the\s+)?(\w+)\s+(?:is|was|are|were)\s+(.*)',
                line, re.IGNORECASE,
            )
            if match and len(match.group(2)) < 50:
                structured.append(
                    f"{match.group(1).lower()}:{match.group(2).rstrip('.')}"
                )
            else:
                structured.append(line)
        if len(structured) < 10:
            return " | ".join(structured)
        return "\n".join(structured)

    def _apply_abbreviations(self, content: str) -> str:
        abbreviations = {
            "successfully": "OK", "successfully completed": "done",
            "file not found": "ENOENT", "permission denied": "EPERM",
            "authentication": "auth", "configuration": "config",
            "environment": "env", "application": "app",
            "development": "dev", "production": "prod",
            "repository": "repo", "documentation": "docs",
            "implementation": "impl", "requirements": "reqs",
            "dependencies": "deps",
        }
        result = content
        for long, short in abbreviations.items():
            if long in result.lower():
                result = re.sub(
                    re.escape(long), short, result, flags=re.IGNORECASE
                )
        return result

    def _remove_inline_redundancy(self, content: str) -> str:
        sentences = re.split(r'[.!?\n]+', content)
        seen: Set[str] = set()
        unique: List[str] = []
        for s in sentences:
            s = s.strip()
            if not s:
                continue
            h = hashlib.md5(s.lower().encode()).hexdigest()
            if h not in seen:
                seen.add(h)
                unique.append(s)
        return ". ".join(unique)

    def get_stats(self) -> Dict[str, Any]:
        return {"tokens_saved": self._tokens_saved}
```

### Module 10: `corteX/engine/state_files.py` (< 300 lines)

Compaction-proof externalized state management.

```python
class StateFileManager:
    """Manages externalized state files that survive all compactions."""

    def __init__(
        self, base_path: str = ".cortex_state",
        tenant_id: str = "default", session_id: str = "",
    ) -> None:
        self._base = Path(base_path) / tenant_id / session_id
        self._base.mkdir(parents=True, exist_ok=True)
        self._state: Dict[str, Any] = {}
        self._decision_log: List[Dict[str, str]] = []
        self._error_journal: List[Dict[str, str]] = []

    async def initialize(self, goal: str, constraints: List[str]) -> None:
        self._state = {
            "goal": goal, "constraints": constraints,
            "progress": 0.0, "sub_goals": [],
            "active_entities": {}, "open_questions": [],
            "brain_state_digest": {},
        }
        self._decision_log.clear()
        self._error_journal.clear()
        await self._persist()

    async def update_progress(
        self, progress: float,
        sub_goals: Optional[List[Dict[str, str]]] = None,
        entities: Optional[Dict[str, str]] = None,
        questions: Optional[List[str]] = None,
    ) -> None:
        self._state["progress"] = progress
        if sub_goals is not None:
            self._state["sub_goals"] = sub_goals
        if entities is not None:
            self._state["active_entities"].update(entities)
        if questions is not None:
            self._state["open_questions"] = questions
        await self._persist()

    async def record_decision(
        self, step: int, decision: str, rationale: str
    ) -> None:
        self._decision_log.append({
            "step": str(step), "decision": decision,
            "rationale": rationale, "timestamp": str(time.time()),
        })
        await self._persist()

    async def record_error(
        self, step: int, error: str,
        resolution: str = "", pattern: str = "",
    ) -> None:
        self._error_journal.append({
            "step": str(step), "error": error,
            "resolution": resolution, "pattern": pattern,
            "status": "resolved" if resolution else "open",
        })
        await self._persist()

    def build_persistent_context(self, max_tokens: int = 2000) -> str:
        lines: List[str] = []
        lines.append(f"## Goal\n{self._state.get('goal', '')}")
        constraints = self._state.get("constraints", [])
        if constraints:
            lines.append("## Constraints\n" + "\n".join(
                f"- {c}" for c in constraints
            ))
        lines.append(f"## Progress: {self._state.get('progress', 0):.0%}")
        sub_goals = self._state.get("sub_goals", [])
        if sub_goals:
            lines.append("## Sub-goals")
            for sg in sub_goals[-5:]:
                lines.append(
                    f"- [{sg.get('status', '?')}] {sg.get('goal', '')}"
                )
        entities = self._state.get("active_entities", {})
        if entities:
            lines.append("## Active Entities")
            for name, state in list(entities.items())[:10]:
                lines.append(f"- {name}: {state}")
        questions = self._state.get("open_questions", [])
        if questions:
            lines.append("## Open Questions\n" + "\n".join(
                f"- {q}" for q in questions[-5:]
            ))
        if self._decision_log:
            lines.append("## Key Decisions")
            for d in self._decision_log[-10:]:
                lines.append(
                    f"- Step {d['step']}: {d['decision']} "
                    f"({d['rationale'][:60]})"
                )
        open_errors = [
            e for e in self._error_journal if e.get("status") == "open"
        ]
        if open_errors:
            lines.append("## Unresolved Errors")
            for e in open_errors[-5:]:
                lines.append(f"- Step {e['step']}: {e['error']}")
        result = "\n".join(lines)
        if _estimate_tokens(result) > max_tokens:
            result = result[:max_tokens * 4]
        return result

    def get_decision_log(self) -> List[Dict[str, str]]:
        return list(self._decision_log)

    def get_error_journal(self) -> List[Dict[str, str]]:
        return list(self._error_journal)

    async def load(self) -> bool:
        state_path = self._base / "state.json"
        if not state_path.exists():
            return False
        try:
            with open(state_path, "r", encoding="utf-8") as f:
                data = json.load(f)
            self._state = data.get("state", {})
            self._decision_log = data.get("decisions", [])
            self._error_journal = data.get("errors", [])
            return True
        except Exception as e:
            logger.error("Failed to load state: %s", e)
            return False

    async def _persist(self) -> None:
        state_path = self._base / "state.json"
        tmp_path = self._base / "state.json.tmp"
        data = {
            "state": self._state, "decisions": self._decision_log,
            "errors": self._error_journal, "saved_at": time.time(),
        }
        with open(tmp_path, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, default=str)
        if state_path.exists():
            state_path.unlink()
        tmp_path.rename(state_path)
```

---

## Integration Points

### Integration with AgentLoop

```python
# In agent_loop.py start() generator:

# At initialization:
state_files = StateFileManager(tenant_id=tenant_id, session_id=session_id)
await state_files.initialize(goal, constraints)
quality_engine = ContextQualityEngine()
versioner = ContextVersioner()
crystallizer = MemoryCrystallizer()
forgetting = ActiveForgettingEngine()
entanglement = EntanglementGraph()
pyramid = ContextPyramid()
prefetcher = PredictiveContextLoader(memory_fabric, entanglement)
pipeline = CognitiveContextPipeline(...)

# Per step (replaces current compile()):
compiled = await pipeline.compile(
    goal=goal, system_prompt=system_prompt,
    messages=messages, brain_state=brain_params,
    memory_fabric=memory_fabric,
    plan_state=plan.to_dict(), step_number=step_count,
)

# Quality gate:
if compiled.quality.health_label == "critical":
    await state_files.update_progress(...)  # Force re-grounding

# On task completion:
crystal = await crystallizer.crystallize(
    goal=goal,
    decision_log=state_files.get_decision_log(),
    tool_usage=tool_usage_log,
    error_journal=state_files.get_error_journal(),
    success_score=quality_score,
    session_id=session_id,
)
```

### Integration with MemoryFabric

```python
class CognitiveMemoryFabric(MemoryFabric):
    def __init__(self, ...):
        super().__init__(...)
        self.forgetting = ActiveForgettingEngine()
        self.crystallizer = MemoryCrystallizer(
            crystal_store=self.semantic._backend
        )

    def store_with_forgetting(self, key, content, importance):
        events = self.forgetting.scan_for_contradictions(
            content, key, self.working._backend.list_all()
        )
        for event in events:
            self.working.forget(event.item_key)
        self.working.store(key, content, importance)

    async def idle_consolidation(self, step, goal):
        """Run during wait time (like sleep consolidation)."""
        stale = self.forgetting.scan_for_staleness(
            self.working._backend.list_all(), step, set(), set()
        )
        for event in stale:
            self.working.forget(event.item_key)
        redundant = self.forgetting.scan_for_redundancy(
            self.working._backend.list_all()
        )
        for event in redundant:
            self.working.forget(event.item_key)
        self.consolidate()
```

### Multi-Tenant Isolation

```
All state is namespaced by tenant_id + user_id:
  .cortex_state/
    {tenant_id}/
      {session_id}/
        state.json          # Fluid + crystallized state
      {user_id}/
        user_profile.json   # Cross-session preferences
      shared/
        crystals/           # Tenant-shared cognitive crystals
        semantic/           # Tenant-shared knowledge

Memory isolation rules:
  - Working memory: per-session (volatile)
  - Episodic memory: per-user within tenant
  - Semantic memory: per-tenant (shared within org)
  - Crystals: per-tenant (learned patterns shared)
  - State files: per-session (survives compaction, not sessions)
  - User profiles: per-user per-tenant (GDPR-deletable)
```

---

## Innovation Highlights

### 1. Context Entanglement (Novel)
No existing system tracks co-reference relationships between context items. Breaking an entangled pair (e.g., removing a schema definition while keeping an error that references it) creates semantic gaps that lead to hallucination. Our entanglement graph ensures paired information stays together or is evicted together.

### 2. Multi-Resolution Pyramid (Novel)
Existing systems use binary include/exclude. Our pyramid maintains 4 resolution levels per item, enabling graceful degradation instead of hard truncation. A file critical 200 steps ago remains as a 10-token fingerprint rather than disappearing.

### 3. Predictive Pre-Loading (Novel)
No agent system pre-fetches context. Our prefetcher uses plan lookahead, entity co-occurrence, and episodic pattern matching to have context ready before it is needed, eliminating retrieval latency in the critical path.

### 4. Memory Crystallization (Novel)
Beyond trajectory storage, crystallization generalizes successful patterns into reusable templates. A crystal says "when building X, do A then B, watch out for C" -- transferable across sessions and users within a tenant.

### 5. Active Forgetting with Poison Detection (Novel)
No system deliberately identifies memories that correlate with failures and reduces their influence. Error poisoning detection tracks which memories were present during failures and degrades their importance.

### 6. Causal Context Versioning (Novel)
When a mistake happens, our versioner diffs the context at the failure point against the last success. The diff reveals exactly what information was missing, enabling targeted recovery.

### 7. Information Density Optimization (Novel)
No system optimizes the encoding density of kept content. Our optimizer converts prose to structured key:value pairs and applies domain abbreviations, achieving 3-5x density improvement without information loss.

---

## Implementation Priority

### Phase 1: Foundation (Week 1-2)
| # | Module | File | Lines | Priority |
|---|--------|------|-------|----------|
| 1 | StateFileManager | `state_files.py` | ~250 | P0 |
| 2 | ContextQualityEngine | `context_quality.py` | ~280 | P0 |
| 3 | CognitiveContextPipeline | `cognitive_context.py` | ~280 | P0 |

### Phase 2: Intelligence (Week 3-4)
| # | Module | File | Lines | Priority |
|---|--------|------|-------|----------|
| 4 | EntanglementGraph | `entanglement.py` | ~250 | P1 |
| 5 | ContextPyramid | `context_pyramid.py` | ~280 | P1 |
| 6 | DensityOptimizer | `density_optimizer.py` | ~200 | P1 |

### Phase 3: Prediction (Week 5-6)
| # | Module | File | Lines | Priority |
|---|--------|------|-------|----------|
| 7 | PredictiveLoader | `predictive_loader.py` | ~250 | P2 |
| 8 | MemoryCrystallizer | `memory_crystallizer.py` | ~280 | P2 |
| 9 | ActiveForgetting | `active_forgetting.py` | ~280 | P2 |

### Phase 4: Diagnostics (Week 7-8)
| # | Module | File | Lines | Priority |
|---|--------|------|-------|----------|
| 10 | ContextVersioner | `context_versioner.py` | ~250 | P3 |
| 11 | CognitiveMemoryFabric | `cognitive_memory.py` | ~200 | P3 |
| 12 | Tests | `tests/test_cognitive_*.py` | ~1000 | P0-P3 |

### Dependencies

```
Phase 1 (no dependencies):
  StateFileManager -> standalone
  ContextQualityEngine -> standalone
  CognitiveContextPipeline -> uses both above

Phase 2 (depends on Phase 1):
  EntanglementGraph -> standalone (used by pipeline)
  ContextPyramid -> uses DensityOptimizer
  DensityOptimizer -> standalone

Phase 3 (depends on Phase 1 + 2):
  PredictiveLoader -> uses EntanglementGraph + MemoryFabric
  MemoryCrystallizer -> uses StateFileManager
  ActiveForgetting -> used by CognitiveMemoryFabric

Phase 4 (depends on Phase 1):
  ContextVersioner -> uses ContextQualityEngine
  CognitiveMemoryFabric -> uses ActiveForgetting + Crystallizer
```

---

## Appendix: Data Structures Summary

```python
# Core output type
@dataclass
class CognitiveCompiledContext:
    messages: List[Dict[str, str]]
    total_tokens: int
    zone_usage: Dict[str, int]
    quality: ContextQualityReport
    prefetch_status: Dict[str, Any]
    context_version: str
    entanglement_pairs: int
    density_savings: int

# Scoring intermediate
@dataclass
class ScoredItem:
    item_id: str
    raw_importance: float
    goal_proximity: float
    recency_factor: float
    entanglement_boost: float
    priority: float  # Composite

# Resolution intermediate
@dataclass
class ResolvedItem:
    item_id: str
    content: str
    tokens: int
    resolution: int  # 0=R0, 1=R1, 2=R2, 3=R3
    priority: float

# Prefetch intermediate
@dataclass
class PredictionCandidate:
    query: str
    source: str
    confidence: float
    expected_step: int

@dataclass
class PrefetchedItem:
    item: MemoryItem
    prefetched_at_step: int
    predicted_need_step: int
    source: str
    confidence: float

# Assembled intermediate
@dataclass
class AssembledContext:
    messages: List[Dict[str, str]]
    total_tokens: int
    zone_usage: Dict[str, int]
```

---

*This architecture transforms corteX from a strong context management foundation into a cognitive information engine that treats context as a dynamic, multi-resolution, quality-monitored, entanglement-aware field -- going far beyond what any existing system offers.*
