---
name: agent-debug
description: Debug TD AI agents by analyzing chat logs with `tdx llm log`. Covers conversation analysis, tool call verification, knowledge base retrieval issues, and cross-referencing with agent definitions. Use when agents behave unexpectedly or users report issues.
---

# tdx Agent Debug - Troubleshooting Agent Behavior

Debug TD AI agents by analyzing conversation logs and identifying issues in agent behavior.

## When to Use

When users report that an agent is not behaving as expected, offer to debug the conversation.

## Debugging Workflow

```bash
# 1. Get chat ID from user
# Ask: "Please provide the chat ID where the issue occurred. You can find it in the web interface chat screen URL (last part of the URL)"

# 2. Retrieve conversation logs and identify the project/agent
tdx llm log <chat-id> --format jsonl
# IMPORTANT: The first line contains project_name and agent_name fields.
# Use these to identify which project/agent this chat belongs to,
# as the chat may be unrelated to the current working directory.

# 3. Analyze the logs to identify issues:
#    - Incorrect tool calls or missing tool invocations
#    - Malformed function arguments
#    - Unexpected reasoning or response patterns
#    - Context/knowledge base retrieval issues
#    - Temperature/reasoning_effort mismatches

# 4. Cross-reference with agent definition
tdx agent pull "<project_name from step 2>" "<agent_name from step 2>"
# Review agent.yml, prompt.md, and related resources
```

## Analysis Checklist

When analyzing conversation logs, systematically check:

1. User messages and agent responses: Verify the agent understood the user's intent correctly
2. Tool calls: Check if tools were called correctly with expected arguments
3. Knowledge base searches: Verify searches returned relevant results and were properly formatted
4. Prompt adherence: Compare agent behavior against prompt.md instructions
5. Model configuration: Check for temperature/reasoning_effort mismatches or other config issues
6. Error patterns: Look for recurring issues or failure modes

## Report Format

Provide findings with:
- Issue summary: Brief description of what went wrong
- Log references: Specific line numbers or message IDs from the chat log
- Root cause: Underlying reason for the unexpected behavior
- Recommendation: Suggested fixes (e.g., update agent.yml, revise prompt.md, modify knowledge base)

## Related Skills

- agent - Building and configuring TD AI agents
- agent-test - Testing agents with validation workflows
