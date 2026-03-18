# Task 3 Plan: The System Agent

## Overview

Add a `query_api` tool to the agent so it can query the deployed backend API and answer questions about:
- Static system facts (framework, ports, status codes)
- Data-dependent queries (item count, scores, completion rates)
- Bug diagnosis (query API, then read source code to find the bug)

## Implementation Plan

### 1. Tool Schema Definition

Add `query_api` to the `TOOLS` list with OpenAI function-calling format:

```python
{
    "type": "function",
    "function": {
        "name": "query_api",
        "description": "Query the backend API to get system information or data",
        "parameters": {
            "type": "object",
            "properties": {
                "method": {"type": "string", "description": "HTTP method (GET, POST, etc.)"},
                "path": {"type": "string", "description": "API path (e.g., /items/, /analytics/completion-rate)"},
                "body": {"type": "string", "description": "Optional JSON request body for POST/PUT requests"}
            },
            "required": ["method", "path"]
        }
    }
}
```

### 2. Tool Implementation

Create `query_api(method, path, body=None)` function:
- Read `LMS_API_KEY` from `.env.docker.secret`
- Read `AGENT_API_BASE_URL` from environment (default: `http://localhost:42002`)
- Use `urllib.request` (built-in, no extra dependencies) to make HTTP requests
- Add `X-API-Key` header for authentication
- Return JSON string with `status_code` and `body`

### 3. System Prompt Update

Update `SYSTEM_PROMPT` to guide the LLM on tool selection:

- **Use `read_file`** for: wiki documentation, source code, configuration files
- **Use `list_files`** for: discovering file structure
- **Use `query_api`** for: runtime data, API status codes, database contents, system facts

### 4. Environment Variables

Load these from environment:
- `LLM_API_KEY`, `LLM_API_BASE`, `LLM_MODEL` — from `.env.agent.secret`
- `LMS_API_KEY` — from `.env.docker.secret`
- `AGENT_API_BASE_URL` — optional, defaults to `http://localhost:42002`

### 5. Testing Strategy

1. Test `query_api` in isolation with a simple GET request
2. Run `uv run run_eval.py` to check all 10 questions
3. Fix failures one by one, adjusting system prompt if needed

## Benchmark Results (updated after LLM API restart)

**Current score: 3/10**

Passing questions:
- Question 0: Wiki lookup (branch protection) ✓
- Question 1: Wiki lookup (SSH connection) ✓  
- Question 2: Source code reading (FastAPI framework) ✓

Failing questions:
- Question 3 (index 3): "List all API router modules..." - LLM gets stuck in list_files loop, doesn't read files consistently
- Questions 4-9: Not yet tested due to question 3 blocking

## Iteration History

### Attempt 1: Basic system prompt
Result: 3/10 - LLM stuck in list_files loop

### Attempt 2: Added "read files after listing" instruction  
Result: 3/10 - LLM still inconsistent

### Attempt 3: Added "IMPORTANT" warning about reading files immediately
Result: 3/10 - LLM behavior varies between runs (non-deterministic)

## Root Cause Analysis

Question 3 failure is due to **LLM non-determinism**, not code bugs:
- The LLM sometimes reads files correctly, sometimes doesn't
- Same question, same prompt → different behavior on different runs
- The model says "I will read files" but doesn't actually call read_file

## Potential Fixes (for future work)

1. **Few-shot examples**: Add example conversations showing correct tool usage
2. **Chain-of-thought prompting**: Ask LLM to explain its reasoning before acting
3. **Different model**: Try a more reliable LLM provider
4. **Reduce max_tokens**: Force shorter, more focused responses
5. **Temperature adjustment**: Lower temperature for more deterministic output

## Lessons Learned

1. **Bearer authentication**: The backend uses `Authorization: Bearer <API_KEY>` header, not `X-API-Key`. This was discovered by reading the backend auth.py code.

2. **Environment variable loading**: Need to load both `.env.agent.secret` (for LLM) and `.env.docker.secret` (for LMS_API_KEY).

3. **Source extraction**: The regex must match both wiki files (`wiki/*.md`) and source files (`backend/app/*.py`).

4. **System prompt is critical but limited**: Even with explicit instructions, the LLM doesn't always follow them. The model's behavior is non-deterministic.

5. **LLM API reliability**: The agent depends on the LLM service being available. Connection errors cause the agent to fail.

6. **Tool implementation vs. LLM behavior**: The `query_api` tool works perfectly when tested directly. The challenge is getting the LLM to use tools correctly.
