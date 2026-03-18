# Task 2 Plan: The Documentation Agent

## Overview

This plan describes how to extend the Task 1 agent with an agentic loop that can use tools (`read_file`, `list_files`) to answer questions by reading the project wiki.

## LLM Provider and Model

- **Provider:** Qwen Code API
- **Model:** `qwen3-coder-plus`
- **Reason:** Strong tool calling capabilities, OpenAI-compatible API

## Tool Definitions

### 1. `read_file`

**Purpose:** Read contents of a file from the project repository.

**Parameters:**
- `path` (string): Relative path from project root (e.g., `wiki/git-workflow.md`)

**Implementation:**
```python
def read_file(path: str) -> str:
    # Validate path is within project directory
    # Read and return file contents
    #bla bla bla
    #bla bla bla
```
#bla bla bla
**Security:**
- Resolve the path using `Path.resolve()`
- Check that the resolved path starts with the project root
- Reject any path containing `..` or absolute paths
- Return error message if file doesn't exist

### 2. `list_files`

**Purpose:** List files and directories at a given path.

**Parameters:**
- `path` (string): Relative directory path from project root (e.g., `wiki`)

**Implementation:**
```python
def list_files(path: str) -> str:
    # Validate path is within project directory
    # List entries and return as newline-separated string
```

**Security:**
- Same path validation as `read_file`
- Only list directories within project root
- Return error message if path is not a directory

## Tool Schemas (OpenAI Function Calling)

Define tool schemas in OpenAI's function calling format:

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the contents of a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Relative path from project root"}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "List files in a directory",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Relative directory path"}
                },
                "required": ["path"]
            }
        }
    }
]
```

## Agentic Loop

The loop structure:

```
1. Send user question + tool definitions to LLM
2. Parse LLM response
3. If response has tool_calls:
   a. Execute each tool
   b. Append results as tool role messages
   c. Loop back to step 1
4. If response has no tool_calls:
   a. Extract answer from message content
   b. Extract source (file reference from context)
   c. Output JSON and exit
5. If 10 tool calls reached → stop and use current answer
```

### Message History Structure

```python
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},
    {"role": "user", "content": question},
    # After tool calls:
    {"role": "assistant", "content": None, "tool_calls": [...]},
    {"role": "tool", "content": tool_result, "tool_call_id": "..."},
    # ... continue until final answer
]
```

## System Prompt Strategy

The system prompt will instruct the LLM to:

1. Use `list_files` to discover wiki files when unsure where to look
2. Use `read_file` to read specific files and find answers
3. Always include a source reference in the final answer (file path + section anchor)
4. Stop after finding the answer (don't make unnecessary tool calls)
5. Maximum 10 tool calls per question

Example system prompt:
```
You are a helpful assistant that answers questions using the project wiki.

You have access to two tools:
- list_files: List files in a directory
- read_file: Read contents of a file

Strategy:
1. Use list_files to discover wiki files if you're unsure where to look
2. Use read_file to read specific files and find the answer
3. When you find the answer, respond with the answer and include the source as: wiki/filename.md#section-anchor
4. Maximum 10 tool calls per question

Always include the source reference in your final answer.
```

## Path Security Implementation

```python
PROJECT_ROOT = Path(__file__).parent.resolve()

def validate_path(path: str) -> Path:
    """Validate that path is within project directory."""
    # Reject absolute paths
    if Path(path).is_absolute():
        raise ValueError("Absolute paths not allowed")
    
    # Reject path traversal
    if ".." in path:
        raise ValueError("Path traversal not allowed")
    
    # Resolve and check
    full_path = (PROJECT_ROOT / path).resolve()
    if not str(full_path).startswith(str(PROJECT_ROOT)):
        raise ValueError("Path outside project directory")
    
    return full_path
```

## Output Format

```json
{
  "answer": "The final answer text",
  "source": "wiki/git-workflow.md#resolving-merge-conflicts",
  "tool_calls": [
    {"tool": "list_files", "args": {"path": "wiki"}, "result": "..."},
    {"tool": "read_file", "args": {"path": "wiki/git-workflow.md"}, "result": "..."}
  ]
}
```

## Testing Strategy

Two regression tests:

1. **Test read_file usage:**
   - Question: "How do you resolve a merge conflict?"
   - Expected: `read_file` in tool_calls, `wiki/git-workflow.md` in source

2. **Test list_files usage:**
   - Question: "What files are in the wiki?"
   - Expected: `list_files` in tool_calls

## Implementation Steps

1. Create `plans/task-2.md` (this file)
2. Add `read_file` and `list_files` functions with path validation
3. Define tool schemas for OpenAI API
4. Implement the agentic loop with message history tracking
5. Update output JSON to include `source` field
6. Update `AGENT.md` documentation
7. Add regression tests
