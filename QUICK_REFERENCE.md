# Dexter Quick Reference Guide

> **For detailed architecture, see [ARCHITECTURE.md](./ARCHITECTURE.md)**

## 🚀 Quick Start

```bash
# Install dependencies
bun install

# Set up API keys
cp env.example .env
# Edit .env with your API keys

# Run
bun start

# Development mode (auto-reload)
bun dev
```

## 📁 Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/agent/agent.ts` | Core agent loop (ReAct pattern) | 312 |
| `src/agent/scratchpad.ts` | Memory/logging system | 177 |
| `src/cli.tsx` | Main UI orchestrator | 210 |
| `src/hooks/useAgentRunner.ts` | Agent execution hook | 236 |
| `src/model/llm.ts` | Multi-provider LLM interface | ~200 |
| `src/tools/finance/financial-search.ts` | Financial data router | ~300 |

## 🔄 Agent Loop Flow

```
1. User Query → CLI
2. Create Agent instance
3. Agent Loop (max 10 iterations):
   ├─ Call LLM with current prompt
   ├─ If tool calls: Execute tools → Add to scratchpad → Continue
   └─ If no tool calls: Generate final answer → Done
4. Stream answer to UI
```

### 🎯 How Dexter Decides to Continue vs Complete

**Key Signal:** Presence/absence of tool calls in LLM response

**Continues when:** LLM returns `tool_calls` array with items
- Means: "I need more data to answer"
- Action: Execute tools → Next iteration

**Completes when:** LLM returns empty `tool_calls` array
- Means: "I have sufficient data to answer"
- Action: Generate final answer with full context

**Prompts guide this decision:**
```typescript
// Iteration prompt explicitly encourages completion
"Review the data above. If you have sufficient information to answer 
the query, respond directly WITHOUT calling any tools. Only call 
additional tools if there are specific data gaps."
```

**Safety:** Max iterations (default: 10) prevents infinite loops

**See:** [ARCHITECTURE.md - Loop Completion Logic](#) for detailed explanation

## 🧩 Key Components

### Agent (`src/agent/agent.ts`)
```typescript
// Create agent
const agent = Agent.create({ model: 'gpt-5.2', maxIterations: 10 });

// Run agent (returns async generator)
for await (const event of agent.run(query)) {
  // Handle: thinking, tool_start, tool_end, answer_chunk, done
}
```

### Scratchpad (`src/agent/scratchpad.ts`)
```typescript
const scratchpad = new Scratchpad(query);

// Add tool result
scratchpad.addToolResult(toolName, args, fullResult, llmSummary);

// Get summaries for iteration (lightweight)
const summaries = scratchpad.getToolSummaries();

// Get full data for final answer (complete)
const fullData = scratchpad.getFullContexts();
```

### Tool Registry (`src/tools/registry.ts`)
```typescript
// Get all tools
const tools = getTools(model);

// Tools are conditionally registered:
- financial_search (always)
- web_search (if TAVILY_API_KEY set)
```

### LLM Interface (`src/model/llm.ts`)
```typescript
// Call LLM
const response = await callLlm(prompt, {
  model: 'gpt-5.2',
  systemPrompt: '...',
  tools: [...],
});

// Stream LLM response
for await (const chunk of streamLlmResponse(prompt, { model })) {
  console.log(chunk);
}
```

## 🛠️ Adding a New Tool

1. **Create tool file** in `src/tools/your-tool/`
```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { z } from 'zod';

export const myNewTool = new DynamicStructuredTool({
  name: 'my_new_tool',
  description: 'What this tool does',
  schema: z.object({
    param1: z.string().describe('Description'),
  }),
  func: async ({ param1 }) => {
    // Implementation
    return 'result';
  },
});
```

2. **Create description** in `src/tools/descriptions/`
```typescript
export const MY_NEW_TOOL_DESCRIPTION = `
What it does...

**When to use:**
- ...

**When NOT to use:**
- ...
`;
```

3. **Register tool** in `src/tools/registry.ts`
```typescript
import { myNewTool } from './your-tool/index.js';
import { MY_NEW_TOOL_DESCRIPTION } from './descriptions/index.js';

export function getToolRegistry(model: string): RegisteredTool[] {
  const tools: RegisteredTool[] = [
    // ... existing tools
    {
      name: 'my_new_tool',
      tool: myNewTool,
      description: MY_NEW_TOOL_DESCRIPTION,
    },
  ];
  return tools;
}
```

## 🎨 Adding a New UI Component

Components use **Ink** (React for CLIs):

```typescript
import React from 'react';
import { Box, Text } from 'ink';

export function MyComponent({ data }: { data: string }) {
  return (
    <Box>
      <Text color="cyan">{data}</Text>
    </Box>
  );
}
```

Add to `src/cli.tsx`:
```typescript
import { MyComponent } from './components/MyComponent.js';

// In render:
<MyComponent data={someData} />
```

## 📊 Event Types

Agent emits these events during execution:

```typescript
type AgentEvent =
  | { type: 'thinking'; message: string }
  | { type: 'tool_start'; tool: string; args: Record<string, unknown> }
  | { type: 'tool_end'; tool: string; result: string; duration: number }
  | { type: 'tool_error'; tool: string; error: string }
  | { type: 'answer_start' }
  | { type: 'answer_chunk'; text: string }
  | { type: 'done'; answer: string; toolCalls: [...]; iterations: number }
```

Handle in `useAgentRunner`:
```typescript
switch (event.type) {
  case 'thinking':
    // Update UI to show thinking state
    break;
  case 'tool_start':
    // Show tool execution started
    break;
  // ...
}
```

## 🔧 Configuration

### Environment Variables

```bash
# LLM Providers (pick one or more)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=AI...
XAI_API_KEY=xai-...

# Local models
OLLAMA_BASE_URL=http://127.0.0.1:11434

# Data sources
FINANCIAL_DATASETS_API_KEY=...  # Required
TAVILY_API_KEY=tvly-...         # Optional
```

### Model Selection

Available models (in `src/model/llm.ts`):
- **OpenAI**: `gpt-5.2`, `gpt-4.1`, `gpt-4-turbo`
- **Anthropic**: `claude-sonnet-4-20250514`, `claude-haiku-4-5`
- **Google**: `gemini-3-flash-preview`, `gemini-2-flash-thinking`
- **xAI**: `grok-4-1-fast-reasoning`
- **Ollama**: Any local model

Switch models with `/model` command in CLI.

## 🐛 Debugging

### Enable Debug Panel
In `src/cli.tsx`:
```typescript
<DebugPanel maxLines={8} show={true} />  // Set to true
```

### View Scratchpad Files
```bash
ls -la .dexter/scratchpad/
cat .dexter/scratchpad/20260122-153045_abc123def456.jsonl
```

Each line is a JSON object:
```json
{"type":"init","content":"What's Apple's revenue?","timestamp":"2026-01-22T15:30:45.123Z"}
{"type":"tool_result","toolName":"financial_search","args":{...},"result":{...},"llmSummary":"...","timestamp":"..."}
{"type":"thinking","content":"I now have the data...","timestamp":"..."}
```

### Agent Execution Logs

The agent emits detailed events. Add logging:
```typescript
for await (const event of agent.run(query)) {
  console.log('Event:', event.type, event);
  // Your handler
}
```

## 📝 Common Tasks

### Change Max Iterations
```typescript
const agent = Agent.create({ maxIterations: 20 }); // Default: 10
```

### Customize System Prompt
In `src/agent/prompts.ts`:
```typescript
export function buildSystemPrompt(model: string): string {
  return `You are Dexter, ...`; // Customize here
}
```

### Add Custom Provider
In `src/model/llm.ts`:
```typescript
const MODEL_PROVIDERS: Record<string, ModelFactory> = {
  'mymodel-': (name, opts) => new ChatMyProvider({ model: name, ...opts }),
  // ...
};
```

### Cancel Running Query
```typescript
// User presses Esc or Ctrl+C
abortController.abort(); // Propagates through entire stack
```

## 🏗️ Architecture Patterns

### Context Compaction
```
❌ Bad: Include full 10KB JSON in each iteration
✅ Good: Use 100-byte summaries during loop, full data at end
```

### Error as Data
```
❌ Bad: Tool error crashes agent
✅ Good: Tool error added to scratchpad, agent adapts
```

### Streaming Everything
```
✅ Tool executions stream start/end events
✅ LLM responses stream chunks
✅ UI updates in real-time
```

### Provider Agnostic
```
✅ Same code works with OpenAI, Anthropic, Google, xAI
✅ Model switching at runtime
✅ Fallback to fast models for summaries
```

## 🧪 Testing

```bash
# Run tests
bun test

# Watch mode
bun test --watch

# Type checking
bun run typecheck
```

Test structure:
```typescript
import { describe, expect, test } from 'bun:test';

describe('MyComponent', () => {
  test('should do something', () => {
    // Your test
    expect(result).toBe(expected);
  });
});
```

## 🚨 Common Issues

### "No tools available"
**Cause:** Missing API key for all providers
**Fix:** Add at least one provider API key to `.env`

### Tool execution fails
**Cause:** Missing API key for data source
**Fix:** Ensure `FINANCIAL_DATASETS_API_KEY` is set

### Slow responses
**Cause:** Using powerful model for everything
**Fix:** Fast models are auto-used for summaries. Consider using a faster primary model.

### Context window exceeded
**Cause:** Too many tool calls or large results
**Fix:** Context compaction should handle this. If still an issue:
- Reduce `maxIterations`
- Use models with larger context windows

## 📚 Key Concepts

### ReAct Pattern
**Re**asoning + **Act**ing in a loop:
1. Think about what's needed
2. Act (use tools)
3. Observe results
4. Repeat until answer is ready

### Scratchpad as Memory
- Single source of truth for all agent work
- Persisted to disk for debugging
- Stores both full data and summaries

### Two-Phase Context
- **Phase 1 (Loop):** Use lightweight summaries
- **Phase 2 (Answer):** Use complete data

### Event-Driven UI
- Agent emits events as generator
- UI subscribes and updates in real-time
- Feels responsive and transparent

## 🔗 Useful Links

- **LangChain Docs**: https://js.langchain.com/docs
- **Ink Docs**: https://github.com/vadimdemedes/ink
- **Bun Docs**: https://bun.sh/docs
- **Zod Docs**: https://zod.dev

## 💡 Pro Tips

1. **Use fast models**: Summaries don't need GPT-5.2
2. **Keep tool results structured**: JSON is easier to parse than text
3. **Write detailed tool descriptions**: They guide the LLM's decisions
4. **Test with simple queries first**: Build up complexity gradually
5. **Monitor scratchpad files**: Great for debugging agent reasoning
6. **Use streaming**: Better UX and insight into what's happening
7. **Handle errors gracefully**: Treat them as data, not crashes

## 🤝 Contributing

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Create a PR with clear description

**Keep PRs small and focused!**

---

For detailed architecture documentation, see [ARCHITECTURE.md](./ARCHITECTURE.md)
