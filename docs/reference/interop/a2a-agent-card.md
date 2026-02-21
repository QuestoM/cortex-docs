# A2A Agent Card API Reference

## Module: `corteX.interop.a2a.agent_card`

Build and parse A2A Agent Cards. An Agent Card is a JSON document served at `/.well-known/agent.json` that describes an A2A agent's capabilities, skills, and supported input/output modes.

---

## Classes

### `AgentSkill`

**Type**: `@dataclass`

A skill advertised by an A2A agent.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | `str` | *(required)* | Unique skill identifier |
| `name` | `str` | *(required)* | Human-readable skill name |
| `description` | `str` | `""` | What this skill does |
| `tags` | `List[str]` | `[]` | Categorization tags |
| `examples` | `List[str]` | `[]` | Example prompts for this skill |

#### Example

```python
from corteX.interop.a2a.agent_card import AgentSkill

skill = AgentSkill(
    id="summarize",
    name="Document Summarization",
    description="Summarizes long documents into key points",
    tags=["nlp", "summarization"],
    examples=["Summarize this quarterly report", "Give me the key takeaways"],
)
```

---

### `AgentCardInfo`

**Type**: `@dataclass`

Parsed representation of an A2A Agent Card.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | *(required)* | Agent display name |
| `description` | `str` | *(required)* | What this agent does |
| `url` | `str` | *(required)* | Base URL where the agent is reachable |
| `version` | `str` | `"1.0"` | Agent version string |
| `skills` | `List[AgentSkill]` | `[]` | Skills this agent provides |
| `input_modes` | `List[str]` | `["text"]` | Supported input modes |
| `output_modes` | `List[str]` | `["text"]` | Supported output modes |

#### Example

```python
from corteX.interop.a2a.agent_card import AgentCardInfo, AgentSkill

card = AgentCardInfo(
    name="Summarizer Agent",
    description="Summarizes documents and extracts key points",
    url="https://summarizer.example.com",
    version="2.0",
    skills=[
        AgentSkill(id="summarize", name="Summarize"),
        AgentSkill(id="extract", name="Extract Key Points"),
    ],
    input_modes=["text"],
    output_modes=["text"],
)
```

---

### `AgentCardBuilder`

Build, parse, and convert A2A Agent Card JSON.

All methods are `@staticmethod` -- no instance state is needed.

#### Methods

##### `build_card`

```python
@staticmethod
def build_card(
    name: str,
    description: str,
    url: str,
    version: str = "1.0",
    tools: Optional[List[Any]] = None,
    skills: Optional[List[AgentSkill]] = None,
) -> Dict[str, Any]
```

Build an A2A Agent Card JSON dict from agent metadata.

**Parameters**:

- `name` (`str`): Agent display name.
- `description` (`str`): What this agent does.
- `url` (`str`): Base URL where this agent is reachable.
- `version` (`str`): Agent version string.
- `tools` (`Optional[List[Any]]`): ToolWrapper-like objects with `.name` and `.description`. Converted to skill entries automatically.
- `skills` (`Optional[List[AgentSkill]]`): Explicit skill list (merged with tools).

**Returns**: `Dict[str, Any]` -- A2A Agent Card as a JSON-serializable dict.

**Example**:

```python
from corteX.interop.a2a.agent_card import AgentCardBuilder, AgentSkill

card = AgentCardBuilder.build_card(
    name="Customer Support Agent",
    description="Handles customer inquiries and ticket management",
    url="https://support.example.com",
    version="1.0",
    skills=[
        AgentSkill(
            id="answer-question",
            name="Answer Question",
            description="Answers customer questions using the knowledge base",
            tags=["support", "qa"],
            examples=["How do I reset my password?"],
        ),
    ],
)

# Result:
# {
#     "name": "Customer Support Agent",
#     "description": "Handles customer inquiries...",
#     "url": "https://support.example.com",
#     "version": "1.0",
#     "skills": [{"id": "answer-question", "name": "Answer Question", ...}],
#     "capabilities": {"input_modes": ["text"], "output_modes": ["text"]},
# }
```

##### `parse_card`

```python
@staticmethod
def parse_card(data: Dict[str, Any]) -> AgentCardInfo
```

Parse an Agent Card JSON dict into `AgentCardInfo`.

**Parameters**:

- `data` (`Dict[str, Any]`): Raw JSON dict (typically from `/.well-known/agent.json`).

**Returns**: `AgentCardInfo` -- Parsed agent card dataclass.

**Example**:

```python
import json

raw = json.loads('{"name": "Writer", "description": "Writes content", "url": "https://writer.example.com", "skills": [{"id": "write", "name": "Write"}]}')

card = AgentCardBuilder.parse_card(raw)
assert card.name == "Writer"
assert len(card.skills) == 1
assert card.skills[0].id == "write"
```

##### `card_to_json`

```python
@staticmethod
def card_to_json(info: AgentCardInfo) -> Dict[str, Any]
```

Convert `AgentCardInfo` back to a JSON-serializable dict matching the A2A Agent Card format.

**Parameters**:

- `info` (`AgentCardInfo`): Parsed agent card info.

**Returns**: `Dict[str, Any]` -- JSON-serializable dict.

**Example**:

```python
# Roundtrip: parse then serialize
card = AgentCardBuilder.parse_card(raw_json)
back_to_json = AgentCardBuilder.card_to_json(card)
assert back_to_json["name"] == raw_json["name"]
```

---

## Agent Card Format

An A2A Agent Card is a JSON document served at `{agent_url}/.well-known/agent.json`:

```json
{
    "name": "My Agent",
    "description": "What this agent does",
    "url": "https://agent.example.com",
    "version": "1.0",
    "skills": [
        {
            "id": "skill-id",
            "name": "Skill Name",
            "description": "What this skill does",
            "tags": ["tag1", "tag2"],
            "examples": ["Example prompt 1"]
        }
    ],
    "capabilities": {
        "input_modes": ["text"],
        "output_modes": ["text"]
    }
}
```

---

## See Also

- [A2A Client](./a2a-client.md) -- Uses `AgentCardBuilder` for discovery
- [A2A Task Bridge](./a2a-task-bridge.md) -- Task lifecycle after discovery
- [Types](./types.md) -- `A2AAgentConfig` dataclass
