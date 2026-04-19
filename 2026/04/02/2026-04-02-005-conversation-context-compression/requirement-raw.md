# Conversation Context Compression (Auto-Compact)

## Requirement Description

Add automatic conversation context compression to vv agent, enabling it to handle long-running conversations and complex multi-step tasks without hitting model context window limits.

Currently, vv agent accumulates messages in conversation history without any management. When the token count approaches the model's context limit, the conversation will fail. Claude Code solves this with a sophisticated 4-stage compression system. vv agent needs at minimum a basic auto-compact mechanism.

## Gap Analysis: Claude Code vs vv Agent

### Claude Code's Context Management (4 stages)

1. **Snip Compact** — Trim excessively long individual messages (e.g., massive tool output)
2. **Micro Compact** — Replace stale tool results with cache markers
3. **Context Collapse** — Fold inactive conversation regions into summaries
4. **Auto Compact** — Full conversation summary when approaching context limits
5. **Reactive Compact** — Emergency compression if API returns 413 (context overflow) error

Additional features:
- Token budget tracking per session
- Protection rules: recent messages and system prompts are never compressed
- User-configurable token budget hints (nudge messages)

### vv Agent's Current State

- The **vage framework** already has `memory/` module with 5 compression variants (fact extraction, summarization, etc.)
- The **session memory** has a configurable `session_memory_token_budget` (default: 4000)
- **Missing**: No mechanism to monitor cumulative conversation token usage, no trigger to compress when approaching limits, no strategy for what to compress vs preserve

### Key Difference

Claude Code treats context management as a **runtime conversation concern** — actively monitoring and compressing the live conversation. vv only has **storage-level compression** for memory persistence, not for the active conversation flow.

## Implementation Suggestion

### MVP Scope (Minimum Viable)

Implement a 2-stage compression approach:

**Stage 1: Tool Output Truncation**
- After each tool execution, check if the tool result exceeds a configurable threshold (e.g., 8000 tokens)
- If exceeded, truncate the output and append a `[truncated]` marker
- Apply to all tools but especially bash, read, grep output

**Stage 2: Auto-Compact (Conversation Summarization)**
- Track cumulative token count of the conversation history
- When approaching model's context limit (e.g., at 80% capacity), trigger auto-compact:
  1. Identify messages eligible for compression (exclude: system prompt, last N user/assistant turns)
  2. Use the existing vage `memory/` compression (summarization variant) to compress eligible messages
  3. Replace eligible messages with a single summary message
  4. Continue the conversation with the compressed history

### Integration Points

1. **Token Counting**: Add a token counter that estimates token usage per message (can use tiktoken or simple heuristic: ~4 chars per token)
2. **Compression Trigger**: Check token budget before each LLM call in the agent main loop
3. **Compression Strategy**: Reuse vage's `memory.Summarizer` or equivalent to summarize old conversation segments
4. **Protected Messages**: System prompt + last 2-4 turns are never compressed
5. **Configuration**: Add `context_compression_threshold` (default: 0.8 = 80% of model's max context) to vv config

### Affected Components

- `vv/cli/` — CLI session needs token tracking
- `vv/dispatches/` — Dispatcher should be context-aware
- `vage/largemodel/` or `vage/agent/` — Core compression logic could live in the framework
- `vv/configs/` — New configuration options

### Complexity Assessment

- **Low-Medium complexity**: The vage framework already has compression primitives
- **Main work**: Wiring token counting and compression trigger into the conversation loop
- **No new external dependencies required**
- **Estimated scope**: ~2-3 files modified, ~200-400 lines of new code

## Impact

- **Without this**: vv agent fails on any conversation exceeding model's context window (~128K-200K tokens), making complex multi-step tasks impossible
- **With this**: vv agent can handle arbitrarily long conversations, enabling real-world usage for complex coding tasks, extended research sessions, and multi-step workflows
- **Enables**: All other features (orchestration, multi-agent, etc.) to work reliably on non-trivial tasks
