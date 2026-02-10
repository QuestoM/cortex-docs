# Build a Code Review Agent

In this tutorial you will build a code review agent that reads source files, analyzes code quality, and produces structured feedback. You will configure the coding context profile for efficient token management, set higher autonomy weights so the agent can work through files independently, and observe how the dual-process router handles simple formatting issues (System 1) versus complex architectural concerns (System 2).

---

## What you will build

- A code review agent with file reading and code analysis tools
- The `coding` context profile for optimal token budget allocation
- Higher autonomy weights for independent operation
- A demonstration of System 1/2 routing on simple vs. complex reviews

## Prerequisites

- Python 3.11+
- corteX installed: `pip install cortex-engine[openai]`
- An OpenAI API key

---

## Step 1: Set up the project

Create a new directory with the agent script and a sample file to review.

```bash
mkdir code-reviewer && cd code-reviewer
touch reviewer.py
touch sample_code.py
```

Add some intentionally flawed code to `sample_code.py` so the agent has something to review:

```python title="sample_code.py"
import os, sys, json
import os  # duplicate import

def process_data(data):
    result = []
    for i in range(len(data)):
        item = data[i]
        if item["status"] == "active":
            if item["value"] > 0:
                if item["category"] == "A":
                    result.append(item["value"] * 2)
                else:
                    result.append(item["value"])
    return result

class UserManager:
    def __init__(self):
        self.users = {}
        self.password = "admin123"  # hardcoded credential

    def add_user(self, name, email):
        self.users[name] = {"email": email, "active": True}

    def get_user(self, name):
        try:
            return self.users[name]
        except:  # bare except
            return None

    def delete_all(self):
        self.users = {}  # no confirmation, no logging
```

---

## Step 2: Define the file reading tool

This tool reads a file from disk and returns its contents. In production, this might read from a Git diff or a code repository API.

```python title="reviewer.py"
import asyncio
import os
import cortex


@cortex.tool(name="read_file", description="Read the contents of a source file")
async def read_file(file_path: str) -> str:
    """
    Read a file and return its contents with line numbers.

    Args:
        file_path: Path to the file to read
    """
    try:
        with open(file_path, "r") as f:
            lines = f.readlines()
        numbered = [f"{i+1:4d} | {line}" for i, line in enumerate(lines)]
        return f"File: {file_path} ({len(lines)} lines)\n" + "".join(numbered)
    except FileNotFoundError:
        return f"Error: File '{file_path}' not found."
```

---

## Step 3: Define the code analysis tool

This tool performs static analysis on a code string and returns structured findings. In production, you might integrate with tools like `ruff`, `pylint`, or `semgrep`.

```python title="reviewer.py"
@cortex.tool(name="analyze_code", description="Run static analysis on Python code")
async def analyze_code(code: str) -> str:
    """
    Analyze Python code for common issues and return findings.

    Args:
        code: The Python source code to analyze
    """
    findings = []

    lines = code.split("\n")
    for i, line in enumerate(lines, 1):
        # Check for bare except
        if "except:" in line and "except:" == line.strip().rstrip():
            findings.append(f"Line {i}: Bare 'except:' clause. Catch specific exceptions.")

        # Check for hardcoded credentials
        for keyword in ["password", "secret", "api_key", "token"]:
            if keyword in line.lower() and "=" in line and '"' in line:
                findings.append(f"Line {i}: Possible hardcoded credential ({keyword}).")

        # Check for deep nesting
        indent = len(line) - len(line.lstrip())
        if indent >= 16 and line.strip().startswith("if "):
            findings.append(f"Line {i}: Deep nesting (depth {indent // 4}). Consider early returns.")

    # Check for duplicate imports
    imports = [l.strip() for l in lines if l.strip().startswith("import ")]
    seen = set()
    for imp in imports:
        if imp in seen:
            findings.append(f"Duplicate import: {imp}")
        seen.add(imp)

    if not findings:
        return "No issues found. Code looks clean."

    return f"Found {len(findings)} issue(s):\n" + "\n".join(f"  - {f}" for f in findings)
```

---

## Step 4: Configure the coding context profile

The `coding` context profile tells the Cortical Context Engine to preserve code snippets during compression while aggressively compressing verbose tool output like build logs.

```python title="reviewer.py"
context_config = cortex.ContextManagementConfig(
    profile="coding",           # (1)!
    token_budget_ratio=0.85,    # Use 85% of the model's context window
)
```

1. The `coding` profile adjusts compression priorities: code blocks are kept at L0 (verbatim) longer, while tool output from shell commands compresses early at L1 (observation masking).

---

## Step 5: Set autonomy and style weights

A code review agent should work independently through files without asking permission at every step. Set autonomy high, verbosity moderate, and formality neutral.

```python title="reviewer.py"
weights = cortex.WeightConfig(
    autonomy=0.7,           # Work through files independently
    verbosity=0.2,          # Moderate detail in reviews
    formality=0.0,          # Neutral tone -- neither casual nor stiff
    initiative=0.6,         # Proactively flag related issues
    explanation_depth=0.6,  # Explain why each issue matters
)
```

!!! tip "Autonomy for batch workflows"
    High autonomy (0.6-0.8) works well for agents that process multiple items in a batch. The agent will proceed through files without asking "Should I continue?" after each one. Keep it below 0.8 to ensure the agent still pauses for genuinely ambiguous decisions.

---

## Step 6: Create the engine and agent

```python title="reviewer.py"
async def main():
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")},
        },
        orchestrator_model="gpt-4o",       # System 2: complex reasoning
        worker_model="gpt-4o-mini",        # System 1: routine checks
    )

    agent = engine.create_agent(
        name="code_reviewer",
        system_prompt=(
            "You are an expert Python code reviewer. "
            "For each file, read the code, run the analyzer, then produce a structured review. "
            "Organize findings by severity: critical, warning, suggestion. "
            "Always explain WHY each issue matters and provide a corrected code example."
        ),
        tools=[read_file, analyze_code],
        goal_tracking=True,
        weight_config=weights,
        context_config=context_config,
    )

    session = agent.start_session(user_id="developer_1")
```

---

## Step 7: Run a simple review (System 1 path)

First, ask the agent to check for basic formatting issues. This is a routine task -- the dual-process router should handle it via System 1 (the fast path using the worker model).

```python title="reviewer.py"
    # Simple review -- expect System 1 (fast path)
    print("--- Simple Review: Formatting Check ---")
    response = await session.run(
        "Check sample_code.py for any duplicate imports."
    )
    print(f"Response: {response.content}")
    print(f"Model used: {response.metadata.model_used}")  # (1)!

    dp_stats = session.get_dual_process_stats()
    print(f"Dual-process stats: {dp_stats}")
```

1. For a simple formatting check, you should see the worker model (e.g., `gpt-4o-mini`) being used, indicating System 1 routing.

Expected output:

```text
--- Simple Review: Formatting Check ---
Response: I found one duplicate import in sample_code.py:

- **Line 2:** `import os` is imported again after already being included
  in the combined import on line 1 (`import os, sys, json`).

**Fix:** Remove line 2 entirely.

Model used: gpt-4o-mini
Dual-process stats: {'system1_count': 1, 'system2_count': 0, 'system2_ratio': 0.0}
```

---

## Step 8: Run a complex review (System 2 path)

Now ask for a thorough security and architecture review. This is a novel, high-stakes request -- the router should escalate to System 2 (the orchestrator model) for deeper reasoning.

```python title="reviewer.py"
    # Complex review -- expect System 2 (slow path)
    print("\n--- Complex Review: Security & Architecture ---")
    response = await session.run(
        "Do a thorough security audit and architecture review of sample_code.py. "
        "Focus on credential handling, error handling patterns, and "
        "the overall design of the UserManager class."
    )
    print(f"Response: {response.content}")
    print(f"Model used: {response.metadata.model_used}")  # (1)!

    dp_stats = session.get_dual_process_stats()
    print(f"Dual-process stats: {dp_stats}")
```

1. For the complex security audit, you should see the orchestrator model (e.g., `gpt-4o`) being used, indicating System 2 routing.

Expected output:

```text
--- Complex Review: Security & Architecture ---
Response: ## Security & Architecture Review: sample_code.py

### Critical

1. **Hardcoded credential (line 18)**
   `self.password = "admin123"` stores a plaintext password in source code.
   This will end up in version control and is a severe security risk.

   **Fix:** Use environment variables or a secrets manager:
   ```python
   self.password_hash = os.environ["USER_MANAGER_PASSWORD_HASH"]
   ```

2. **Bare except clause (line 26)**
   `except:` catches all exceptions including KeyboardInterrupt and SystemExit.
   This hides bugs and makes debugging difficult.

   **Fix:** Catch the specific exception:
   ```python
   except KeyError:
       return None
   ```

### Warning

3. **Deep nesting in process_data (lines 8-12)**
   Three levels of nested `if` statements reduce readability.

   **Fix:** Use early returns or guard clauses:
   ```python
   def process_data(data):
       return [
           item["value"] * 2 if item["category"] == "A" else item["value"]
           for item in data
           if item["status"] == "active" and item["value"] > 0
       ]
   ```

### Suggestion

4. **delete_all lacks safety guards (line 30)**
   The method silently deletes all users with no confirmation or audit trail.
   Add logging and consider requiring explicit confirmation.

Model used: gpt-4o
Dual-process stats: {'system1_count': 1, 'system2_count': 1, 'system2_ratio': 0.5}
```

!!! info "System 1 vs. System 2 in practice"
    The dual-process router evaluates novelty, complexity, and risk for each request. The simple formatting check matched a familiar pattern (System 1), while the security audit involved novel reasoning about multiple interacting concerns (System 2). In a production code review pipeline processing hundreds of files, this routing saves significant cost and latency by using the cheaper worker model for routine checks.

---

## Step 9: Inspect final brain state and close

```python title="reviewer.py"
    # Final inspection
    print("\n--- Final Brain State ---")
    print(f"Weights: {session.get_weights()}")
    print(f"Goal progress: {session.get_goal_progress()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")

    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Step 10: Run the complete agent

```bash
export OPENAI_API_KEY="sk-..."
python reviewer.py
```

---

## Complete code

```python title="reviewer.py"
import asyncio
import os
import cortex


@cortex.tool(name="read_file", description="Read the contents of a source file")
async def read_file(file_path: str) -> str:
    """Read a file and return its contents with line numbers."""
    try:
        with open(file_path, "r") as f:
            lines = f.readlines()
        numbered = [f"{i+1:4d} | {line}" for i, line in enumerate(lines)]
        return f"File: {file_path} ({len(lines)} lines)\n" + "".join(numbered)
    except FileNotFoundError:
        return f"Error: File '{file_path}' not found."


@cortex.tool(name="analyze_code", description="Run static analysis on Python code")
async def analyze_code(code: str) -> str:
    """Analyze Python code for common issues and return findings."""
    findings = []
    lines = code.split("\n")
    for i, line in enumerate(lines, 1):
        if "except:" in line and "except:" == line.strip().rstrip():
            findings.append(f"Line {i}: Bare 'except:' clause.")
        for kw in ["password", "secret", "api_key", "token"]:
            if kw in line.lower() and "=" in line and '"' in line:
                findings.append(f"Line {i}: Possible hardcoded credential ({kw}).")
        indent = len(line) - len(line.lstrip())
        if indent >= 16 and line.strip().startswith("if "):
            findings.append(f"Line {i}: Deep nesting (depth {indent // 4}).")
    imports = [l.strip() for l in lines if l.strip().startswith("import ")]
    seen = set()
    for imp in imports:
        if imp in seen:
            findings.append(f"Duplicate import: {imp}")
        seen.add(imp)
    if not findings:
        return "No issues found."
    return f"Found {len(findings)} issue(s):\n" + "\n".join(f"  - {f}" for f in findings)


context_config = cortex.ContextManagementConfig(profile="coding", token_budget_ratio=0.85)
weights = cortex.WeightConfig(
    autonomy=0.7, verbosity=0.2, formality=0.0, initiative=0.6, explanation_depth=0.6,
)


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )
    agent = engine.create_agent(
        name="code_reviewer",
        system_prompt=(
            "You are an expert Python code reviewer. "
            "Read the code, run the analyzer, then produce a structured review. "
            "Organize by severity: critical, warning, suggestion. "
            "Explain WHY each issue matters and provide corrected code."
        ),
        tools=[read_file, analyze_code],
        goal_tracking=True,
        weight_config=weights,
        context_config=context_config,
    )
    session = agent.start_session(user_id="developer_1")

    print("--- Simple Review: Formatting Check ---")
    r1 = await session.run("Check sample_code.py for any duplicate imports.")
    print(f"Response: {r1.content}")
    print(f"Model used: {r1.metadata.model_used}")
    print(f"Dual-process: {session.get_dual_process_stats()}")

    print("\n--- Complex Review: Security & Architecture ---")
    r2 = await session.run(
        "Do a thorough security audit and architecture review of sample_code.py. "
        "Focus on credential handling, error handling, and the UserManager design."
    )
    print(f"Response: {r2.content}")
    print(f"Model used: {r2.metadata.model_used}")
    print(f"Dual-process: {session.get_dual_process_stats()}")

    print("\n--- Final Brain State ---")
    print(f"Weights: {session.get_weights()}")
    print(f"Goal progress: {session.get_goal_progress()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")

    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## What you learned

- How to configure the `coding` context profile for efficient token management in code-heavy sessions
- How to set autonomy weights for agents that work independently through batches of items
- How the dual-process router automatically routes simple tasks to System 1 (worker model) and complex tasks to System 2 (orchestrator model)
- How to observe routing decisions through `get_dual_process_stats()`

## Next steps

- [Build a Research Agent](research-agent.md) -- learn about memory fabric and long-running sessions
- [Set Up Multi-Provider Failover](multi-provider.md) -- add resilience with multiple LLM providers
- [Deploy to Production](production.md) -- production-ready enterprise configuration
