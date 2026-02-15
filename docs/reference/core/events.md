# EventBus API Reference

## Module: `corteX.core.events`

The EventBus provides asynchronous pub/sub event dispatching with **per-instance tenant isolation**. Each `EventBus()` instance maintains its own subscriber registry, ensuring that tenants sharing a process cannot leak events to each other.

A module-level default instance provides backward compatibility for single-tenant usage.

## Classes

### `EventType`

**Type**: `str, Enum`

Enumeration of event types that can be published and subscribed to.

#### Values

| Value | String | Description |
|-------|--------|-------------|
| `SYSTEM_STARTUP` | `"system.startup"` | Engine or session initialization completed. |
| `AGENT_THOUGHT` | `"agent.thought"` | Agent intermediate reasoning step. |
| `AGENT_ACTION` | `"agent.action"` | Agent executed an action (tool call, decision). |
| `ARTIFACT_CREATED` | `"artifact.created"` | A new artifact was produced (code, document, plan). |
| `ERROR_CRITICAL` | `"error.critical"` | A critical error occurred. |
| `USER_INTERACTION` | `"user.interaction"` | User input or feedback received. |

#### Example

```python
from corteX.core.events import EventType

# Use as subscription filter
event_type = EventType.AGENT_ACTION

# String comparison works
assert event_type == "agent.action"
```

---

### `Event`

**Type**: `pydantic.BaseModel`

An event published on the bus.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | `str` | Auto-generated UUID | Unique event identifier. |
| `type` | `EventType` | (required) | The event type. |
| `timestamp` | `datetime` | `datetime.now()` | When the event was created. |
| `source` | `str` | (required) | Identifier of the component that published the event (e.g., `"session"`, `"tool_executor"`). |
| `payload` | `Dict[str, Any]` | (required) | Event-specific data. |

#### Example

```python
from corteX.core.events import Event, EventType

event = Event(
    type=EventType.AGENT_ACTION,
    source="session_abc123",
    payload={
        "tool_name": "search_docs",
        "arguments": {"query": "return policy"},
        "success": True,
    },
)
```

---

### `EventHandler`

**Type**: `Protocol`

Protocol for event handler callables. Any async function accepting an `Event` satisfies this protocol.

```python
class EventHandler(Protocol):
    async def __call__(self, event: Event) -> None: ...
```

---

### `EventBus`

Asynchronous pub/sub with per-instance subscriber isolation.

Each `EventBus()` owns its own subscriber dict. Two EventBus instances never share state -- this is the critical tenant-isolation guarantee.

#### Constructor

```python
EventBus(*, name: Optional[str] = None)
```

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `Optional[str]` | `"default"` | Human-readable name for this bus instance (used in logging). |

---

#### Instance Methods (Tenant-Isolated)

##### `subscribe_instance`

```python
def subscribe_instance(
    self,
    event_type: EventType,
    handler: EventHandler,
) -> None
```

Subscribe a handler on THIS instance only. The handler will be called whenever an event of the given type is published on this specific bus.

**Parameters**:

- `event_type` (`EventType`): The event type to listen for.
- `handler` (`EventHandler`): An async callable to invoke when the event is published.

##### `unsubscribe_instance`

```python
def unsubscribe_instance(
    self,
    event_type: EventType,
    handler: EventHandler,
) -> bool
```

Remove a handler from this instance. Returns `True` if the handler was found and removed, `False` otherwise.

##### `publish_instance`

```python
async def publish_instance(self, event: Event) -> None
```

Publish an event to subscribers of THIS instance only. All matching handlers are called concurrently via `asyncio.gather`.

**Parameters**:

- `event` (`Event`): The event to publish.

##### `subscriber_count`

```python
def subscriber_count(
    self,
    event_type: Optional[EventType] = None,
) -> int
```

Return the number of subscribers, optionally filtered to a specific event type.

##### `clear_subscribers`

```python
def clear_subscribers(self) -> None
```

Remove all subscribers from this instance.

##### `name` (property)

```python
@property
def name(self) -> str
```

The human-readable name of this bus instance.

---

#### Class Methods (Legacy / Global Bus)

These methods delegate to the module-level default `EventBus` instance. Useful for single-tenant applications or backward compatibility.

##### `EventBus.subscribe`

```python
@classmethod
def subscribe(cls, event_type: EventType, handler: EventHandler) -> None
```

Subscribe on the default (global) bus.

##### `EventBus.publish`

```python
@classmethod
async def publish(cls, event: Event) -> None
```

Publish on the default (global) bus.

##### `EventBus.get_default`

```python
@classmethod
def get_default(cls) -> EventBus
```

Return the module-level default EventBus instance.

---

## Module-Level Instances

```python
# Module-level default instance
bus = EventBus(name="global-default")
```

The `bus` variable is the default EventBus instance, available as `corteX.core.events.bus`.

---

## Usage Examples

### Single-Tenant (Global Bus)

```python
from corteX.core.events import EventBus, Event, EventType

# Define handler
async def on_action(event: Event):
    print(f"Action: {event.source} -> {event.payload}")

# Subscribe to global bus
EventBus.subscribe(EventType.AGENT_ACTION, on_action)

# Publish to global bus
await EventBus.publish(Event(
    type=EventType.AGENT_ACTION,
    source="my_agent",
    payload={"tool": "search", "result": "found 5 items"},
))
```

### Multi-Tenant (Isolated Buses)

```python
from corteX.core.events import EventBus, Event, EventType

# Each tenant gets their own bus
tenant_a_bus = EventBus(name="tenant-a")
tenant_b_bus = EventBus(name="tenant-b")

async def tenant_a_handler(event: Event):
    print(f"[Tenant A] {event.payload}")

async def tenant_b_handler(event: Event):
    print(f"[Tenant B] {event.payload}")

# Subscribe to isolated buses
tenant_a_bus.subscribe_instance(EventType.AGENT_ACTION, tenant_a_handler)
tenant_b_bus.subscribe_instance(EventType.AGENT_ACTION, tenant_b_handler)

# Publishing to tenant A does NOT trigger tenant B's handler
await tenant_a_bus.publish_instance(Event(
    type=EventType.AGENT_ACTION,
    source="agent_a",
    payload={"data": "sensitive_to_a"},
))
```

### Observability Dashboard

```python
from corteX.core.events import EventBus, Event, EventType

bus = EventBus(name="observability")

# Track all agent actions for metrics
action_log = []

async def log_action(event: Event):
    action_log.append({
        "timestamp": event.timestamp.isoformat(),
        "source": event.source,
        "tool": event.payload.get("tool_name", "N/A"),
        "success": event.payload.get("success", True),
    })

bus.subscribe_instance(EventType.AGENT_ACTION, log_action)
bus.subscribe_instance(EventType.ERROR_CRITICAL, log_action)

# Later: query the log
print(f"Total actions: {len(action_log)}")
print(f"Subscribers: {bus.subscriber_count()}")
```

---

## Performance Notes

- All handlers for a given event type execute concurrently via `asyncio.gather`
- No I/O is performed by the EventBus itself -- it is purely an in-memory dispatcher
- Typical dispatch latency: <0.1ms for up to 100 handlers per event type
- The `_publish_count` attribute tracks total events published for monitoring

---

## See Also

- [Contracts API Reference](./contracts.md) - Core data types and protocols
- [Registry API Reference](./registry.md) - Plugin registry
- [Observability Concept Guide](../concepts/observability.md) - Observability architecture
- [Security & Isolation](../../enterprise/security.md) - Tenant isolation design
