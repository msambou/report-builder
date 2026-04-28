# Document Assistant — Implementation Documentation

A multi-agent document processing system built with LangChain and LangGraph. It classifies user intent and routes requests to specialized agents that can answer questions, summarize documents, and perform calculations on financial and healthcare records.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Project Structure](#project-structure)
3. [Implementation Decisions](#implementation-decisions)
4. [How State and Memory Works](#how-state-and-memory-works)
5. [How Structured Outputs Are Enforced](#how-structured-outputs-are-enforced)
6. [Example Conversations](#example-conversations)

---

## Getting Started

### Prerequisites

- Python 3.9+
- OpenAI API key

### Installation

```bash
# 1. Clone the repository and navigate into the solution directory
cd solution

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set up your environment variables
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY
```

### Running the Assistant

```bash
python main.py
```

You will be prompted for a user ID, then you can start sending messages. Available commands:

| Command | Description |
|---------|-------------|
| `/docs` | List all available documents |
| `/help` | Show example queries |
| `/quit` | Exit |

---

## Project Structure

```
solution/
├── src/
│   ├── __init__.py     # Package marker
│   ├── schemas.py      # Pydantic models for structured I/O
│   ├── retrieval.py    # In-memory document retrieval engine
│   ├── tools.py        # LangChain tools (calculator, search, reader)
│   ├── prompts.py      # Prompt templates for each agent
│   ├── agent.py        # LangGraph workflow and all node functions
│   └── assistant.py    # DocumentAssistant class and session management
├── sessions/           # Created at runtime — persisted session files (JSON)
├── logs/               # Created at runtime — tool usage logs (JSON)
├── main.py             # Interactive CLI entry point
├── requirements.txt
└── .env.example        # Environment variable template
```

---

## Implementation Decisions

### 1. Multi-Agent Architecture with LangGraph

Rather than using a single monolithic prompt, the system routes each user request to a dedicated agent based on intent. This makes each agent's prompt focused and avoids confusing the model with instructions irrelevant to the task.

The workflow graph looks like this:

```
[User Input]
     │
     ▼
classify_intent
     │
     ├── "qa"            ──▶ qa_agent
     ├── "summarization" ──▶ summarization_agent
     └── "calculation"   ──▶ calculation_agent
                                   │
                              (all three)
                                   │
                                   ▼
                            update_memory
                                   │
                                   ▼
                                 END
```

### 2. ReAct Agents for Tool Use

Each specialized agent (`qa_agent`, `summarization_agent`, `calculation_agent`) uses LangGraph's `create_react_agent`. This allows the LLM to reason about which tools to call, call them, observe the result, and reason again before producing a final answer — rather than calling tools blindly in a single step.

### 3. SimulatedRetriever (No Vector Database)

Document retrieval is handled by `SimulatedRetriever` in `retrieval.py`. It supports keyword search, type-based filtering, and amount-range queries using simple string matching and metadata comparisons. This avoids the overhead of a vector database while still demonstrating realistic retrieval patterns.

### 4. Calculator Tool Safety

The calculator tool in `tools.py` validates the input expression with a regex before calling `eval()`. Only digits, basic operators (`+ - * / % ^`), parentheses, decimal points, and spaces are permitted. Any expression containing letters or unexpected characters is rejected with an error string, preventing code injection. The tool always returns a plain `str` to satisfy the LangChain tool contract; full metadata (expression, result, explanation) is written to the `ToolLogger` separately for observability.

```python
if not re.match(r'^[\d\s\+\-\*\/\.\(\)\%\^]+$', mathematical_expression):
    # reject — return error string, log the attempt
    return f"Error: Unsafe expression rejected: `{mathematical_expression}`."
```

### 5. Session Storage

Each session is serialized to a JSON file under `./sessions/{session_id}.json`. On resumption, the file is loaded and the prior document context is restored. This allows a user to pick up a conversation in a new process invocation.

---

## How State and Memory Works

### AgentState

`AgentState` (defined in `agent.py`) is a `TypedDict` that flows through every node in the LangGraph graph. It carries:

| Field | Purpose |
|-------|---------|
| `user_input` | The raw text the user typed this turn |
| `messages` | Full message history, merged with `add_messages` reducer |
| `intent` | The classified `UserIntent` object |
| `next_step` | Routing signal read by `should_continue` |
| `conversation_summary` | Running text summary produced by `update_memory` |
| `active_documents` | Document IDs referenced across the conversation |
| `current_response` | The raw output from the current agent node |
| `tools_used` | Names of tools called during this turn |
| `actions_taken` | Accumulated list of node names that have executed |
| `session_id` / `user_id` | Session identifiers |

### State Reducers

Two fields use reducers so that updates from different nodes are merged rather than overwritten:

- **`messages`** uses `add_messages` — new messages are appended to the history, not replaced.
- **`actions_taken`** uses `operator.add` — each node appends its name to the list, so the final value records the full execution path for the turn (e.g. `["classify_intent", "calculation_agent", "update_memory"]`).

### InMemorySaver Checkpointer

The workflow is compiled with `InMemorySaver()`:

```python
memory = InMemorySaver()
return workflow.compile(checkpointer=memory)
```

The checkpointer snapshots `AgentState` after every node. When `workflow.invoke` is called again with the same `thread_id`, LangGraph restores the previous snapshot and merges it with the new input. This means the assistant remembers all prior messages and context within a session without any manual history management in application code.

The `thread_id` is set to `self.current_session.session_id` in `process_message`, so each user session gets its own isolated checkpoint stream.

### update_memory Node

After every agent turn, `update_memory` runs. It reads the current message history, sends it to the LLM with a summarization prompt, and writes back:

- `conversation_summary` — a condensed plain-text summary of the conversation so far.
- `active_documents` — document IDs the LLM identified as relevant in this turn, merged with any previously tracked IDs.

This summary is injected into the intent classification prompt on the next turn, giving the classifier awareness of prior context.

---

## How Structured Outputs Are Enforced

Every LLM call in this system returns a typed Pydantic object rather than a raw string. This is achieved with `llm.with_structured_output(Schema)` or `response_format=Schema` in the ReAct agent.

### Schemas and Where They Are Used

| Schema | Fields | Used In |
|--------|--------|---------|
| `UserIntent` | `intent_type`, `confidence` (float 0–1), `reasoning` | `classify_intent` — routes the graph |
| `AnswerResponse` | `question`, `answer`, `sources` (default `[]`), `confidence` (float 0–1, default `0.0`), `timestamp` (default now) | `qa_agent` |
| `SummarizationResponse` | `summary`, `key_points`, `original_length`, `document_ids`, `timestamp` | `summarization_agent` |
| `CalculationResponse` | `expression`, `result`, `explanation`, `units`, `timestamp` | `calculation_agent` |
| `UpdateMemoryResponse` | `summary`, `document_ids` | `update_memory` |

### Intent Classification

```python
parser = llm.with_structured_output(UserIntent)
chain = prompt | parser
response = chain.invoke({"user_input": ..., "conversation_history": ...})
# response is guaranteed to be a UserIntent object
```

### Agent Nodes

Each agent node uses `create_react_agent` with `response_format=Schema`, which instructs the model to emit a final structured response matching the schema after completing its tool calls:

```python
agent = create_react_agent(
    model=llm_with_tools,
    tools=tools,
    response_format=SummarizationResponse,  # enforces structured final answer
)
```

This means the assistant's final answer always contains typed, validated fields — not free-form text — making the output predictable and easy to display or process downstream.

---

## Example Conversations

### Example 1 — Q&A

**Input:**
```
What are the payment terms for invoice INV-002?
```

**Output:**
```
The payment terms for invoice INV-002 (Invoice #12346, issued to TechStart Inc.)
are Net 45 days.

INTENT: qa
TOOLS USED: document_reader
```

**What happened:** `classify_intent` identified this as a `qa` request. `qa_agent` called `document_reader` with `doc_id=INV-002`, extracted the relevant line, and returned a structured `AnswerResponse`.

---

### Example 2 — Summarization

**Input:**
```
Summarize all contracts
```

**Output:**
```
### Contract Summary

**Document ID:** CON-001
**Title:** Service Agreement
**Parties:** DocDacity Solutions Inc. (Provider) and Healthcare Partners LLC (Client)
**Services:** Document Processing Platform Access, 24/7 Technical Support,
              Monthly Data Analytics Reports, Compliance Monitoring
**Duration:** 12 months
**Monthly Fee:** $15,000 | **Total Value:** $180,000
**Termination:** 60 days written notice by either party

INTENT: summarization
TOOLS USED: document_search, document_reader
CONVERSATION SUMMARY: The user requested a summary of all contracts. One contract
was found (CON-001), a service agreement between DocDacity Solutions Inc. and
Healthcare Partners LLC valued at $180,000 over 12 months.
```

**What happened:** `classify_intent` routed to `summarization_agent`. The agent called `document_search` to find contracts, then `document_reader` to get the full content, and returned a `SummarizationResponse` with `key_points` and `document_ids`. `update_memory` produced a conversation summary.

---

### Example 3 — Calculation

**Input:**
```
What is the total value of all invoices combined?
```

**Output:**
```
The total value of all invoices is $305,800.

Calculation:
- INV-001: $22,000
- INV-002: $69,300
- INV-003: $214,500
- Expression: 22000 + 69300 + 214500 = 305800

INTENT: calculation
TOOLS USED: document_search, document_reader, calculator_tool
```

**What happened:** `classify_intent` routed to `calculation_agent`. The agent used `document_search` to find all invoices, `document_reader` to read each one and extract totals, then called `calculator_tool` with the expression `22000 + 69300 + 214500`. All arithmetic was performed by the tool — the LLM did not calculate mentally. The result was returned as a `CalculationResponse`.

---

### Example 4 — Multi-Turn Conversation (Memory in Action)

**Turn 1:**
```
User:      What is the claimant's name on claim CLM-001?
Assistant: The claimant on claim CLM-001 is John Doe.
           INTENT: qa | TOOLS USED: document_reader
```

**Turn 2:**
```
User:      What was the total amount they claimed?
Assistant: John Doe's total claim amount on CLM-001 is $2,450,
           covering a hospital visit, diagnostic tests, medication,
           and a follow-up consultation.
           INTENT: qa | TOOLS USED: document_reader
```

**What happened:** On Turn 2, the user said "they" without specifying who. Because the `InMemorySaver` checkpointer preserved the message history under the same `thread_id`, the LLM had full context from Turn 1 and correctly resolved "they" to John Doe without being told again.

---

## Built With

- [LangChain](https://python.langchain.com/) `>=0.2.0` — prompt templates, tool decorators, structured output
- [LangGraph](https://langchain-ai.github.io/langgraph/) `>=0.6.7` — stateful multi-agent workflow with checkpointing
- [langchain-openai](https://python.langchain.com/docs/integrations/chat/openai/) `>=0.1.0` — OpenAI chat model integration
- [langchain-core](https://python.langchain.com/docs/concepts/) `>=0.2.0` — messages, prompts, runnables
- [OpenAI GPT-4o](https://platform.openai.com/) via `openai>=1.0.0` — underlying language model
- [Pydantic](https://docs.pydantic.dev/) `>=2.0.0` — schema validation and structured output enforcement
- [python-dotenv](https://pypi.org/project/python-dotenv/) `>=1.0.0` — environment variable loading
- [print-color](https://pypi.org/project/print-color/) `>=0.4.6` — colored CLI output
