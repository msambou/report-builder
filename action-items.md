# Assignment Action Items

Items that still need attention before submission.

---

## 1. schemas.py — Task 1.2: UserIntent `confidence` type

**File:** `starter/src/schemas.py` line 58

The `confidence` field is typed as `int` but the rubric requires a `float between 0 and 1`.

**Fix:**
```python
# Change:
confidence: int
# To:
confidence: float = Field(ge=0.0, le=1.0, description="Confidence score between 0 and 1")
```

---

## 2. tools.py — Task 4.1: Calculator tool missing safety validation and logger call

**File:** `starter/src/tools.py` lines 71–104

The rubric requires:
1. Validating the expression for safety (only allow basic math operations) before calling `eval()`.
2. Logging tool usage with the `ToolLogger`.

Currently `eval()` is called with no validation, and `logger` is never used inside the tool.

**Fix — add validation and logger call:**
```python
@tool
def calculator_tool(mathematical_expression: str) -> dict:
    """..."""
    # Safety: only allow digits, operators, spaces, parentheses, and dots
    if not re.match(r'^[\d\s\+\-\*\/\.\(\)\%\^]+$', mathematical_expression):
        result = {
            "expression": mathematical_expression,
            "result": float("nan"),
            "explanation": f"Unsafe expression rejected: `{mathematical_expression}`.",
            "units": None,
            "timestamp": datetime.now(),
        }
        logger.log_tool_use("calculator_tool", {"expression": mathematical_expression}, result)
        return result
    try:
        value = eval(mathematical_expression)
        result = {
            "expression": mathematical_expression,
            "result": value,
            "explanation": f"Evaluated `{mathematical_expression}` to get {value}.",
            "units": None,
            "timestamp": datetime.now(),
        }
    except Exception as e:
        result = {
            "expression": mathematical_expression,
            "result": float("nan"),
            "explanation": f"Could not evaluate `{mathematical_expression}`. Error: {e}",
            "units": None,
            "timestamp": datetime.now(),
        }
    logger.log_tool_use("calculator_tool", {"expression": mathematical_expression}, result)
    return result
```

---

## 3. agent.py — Task 2.3: summarization_agent and calculation_agent use wrong response schema

**File:** `starter/src/agent.py` lines 168 and 195

Both agents call `invoke_react_agent(AnswerResponse, ...)` but the rubric says to use the schema that corresponds to each node:
- `summarization_agent` → `SummarizationResponse`
- `calculation_agent` → `CalculationResponse`

**Fix:**
```python
# summarization_agent (line 168):
result, tools_used = invoke_react_agent(SummarizationResponse, messages, llm, tools)

# calculation_agent (line 195):
result, tools_used = invoke_react_agent(CalculationResponse, messages, llm, tools)
```

---

## 4. assistant.py — Task 2.6: `thread_id` set to `user_input` instead of `session_id`

**File:** `starter/src/assistant.py` line 123

The `thread_id` in the config must be the session ID so the checkpointer correctly persists state per session.

**Fix:**
```python
# Change:
"thread_id": user_input,
# To:
"thread_id": self.current_session.session_id,
```

---

## 5. README.md — Fill in project README

**File:** `README.md` (project root)

The root README is still the blank template. The rubric/submission checklist typically requires a completed README with project title, description, dependencies, installation steps, and usage instructions.

Fill in the README based on the `starter/README.md` content, covering:
- Project title and description
- Prerequisites (Python 3.9+, OpenAI API key)
- Installation steps (`pip install -r requirements.txt`, `.env` setup)
- How to run (`python main.py`)
- Brief description of the agent architecture

---

## Summary Table

| # | File | Task | Status |
|---|------|------|--------|
| 1 | `schemas.py:58` | UserIntent `confidence` must be `float`, not `int` | ❌ Fix needed |
| 2 | `tools.py:71` | Calculator: add safety validation + logger call | ❌ Fix needed |
| 3 | `agent.py:168,195` | Use `SummarizationResponse` / `CalculationResponse` (not `AnswerResponse`) | ❌ Fix needed |
| 4 | `assistant.py:123` | Set `thread_id` to `self.current_session.session_id` | ❌ Fix needed |
| 5 | `README.md` | Complete the project README | ❌ Fill in |
