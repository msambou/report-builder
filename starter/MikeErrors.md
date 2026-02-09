# Errors encountered

## Post-ToDo Completion Errors

## Import Errors
After completing all To Dos and running `python main.py`.

```shell
    from src.assistant import DocumentAssistant
  File "/Users/mikesambou/projects/udacity/report-builder/starter/src/assistant.py", line 13, in <module>
    from agent import create_workflow, AgentState
  File "/Users/mikesambou/projects/udacity/report-builder/starter/src/agent.py", line 16, in <module>
    from prompts import get_intent_classification_prompt, get_chat_prompt_template, MEMORY_SUMMARY_PROMPT
  File "/Users/mikesambou/projects/udacity/report-builder/starter/src/prompts.py", line 1, in <module>
    from langchain.prompts import PromptTemplate, ChatPromptTemplate, MessagesPlaceholder
ModuleNotFoundError: No module named 'langchain.prompts'
```

### Resolution
* I had to import the PromptTemplate, ChatPromptTemplate, MessagesPlaceholder from `langchain_core` instead of `langchain`

## Edge Error

```shell
  ^
  File "/Users/mikesambou/projects/udacity/report-builder/.venv/lib/python3.13/site-packages/langgraph/graph/state.py", line 1025, in validate
    raise ValueError(f"Found edge ending at unknown node `{target}`")
ValueError: Found edge ending at unknown node `<function update_memory at 0x10cfc3f60>`
```

### Resolution
- I narrowed down the error to the agent.py file.
- The lines where I added the edges to the workflow, I made sure to put the `update_memory` in quotes


## Function Error
```shell
Error: Unsupported function

None

Functions must be passed in as Dict, pydantic.BaseModel, or Callable. If they're a dict they must either be in OpenAI function format or valid JSON schema with top-level 'title' key.
```

