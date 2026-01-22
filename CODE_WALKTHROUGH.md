# Dexter Code Walkthrough

> **A step-by-step walkthrough of how Dexter processes a query with annotated code**

This document follows a real query through the entire system, highlighting the exact code that executes at each step.

## Example Query

**User asks:** "What's Apple's revenue for the last 4 quarters?"

Let's trace how Dexter processes this query.

---

## Step 1: User Input (CLI Layer)

### File: `src/cli.tsx`

```typescript
// User types in the Input component
export function CLI() {
  const handleSubmit = useCallback(async (query: string) => {
    // 1️⃣ User submits: "What's Apple's revenue for the last 4 quarters?"
    
    // Handle special commands
    if (query.toLowerCase() === 'exit' || query.toLowerCase() === 'quit') {
      exit();
      return;
    }
    
    if (query === '/model') {
      startSelection();
      return;
    }
    
    // 2️⃣ Save to input history (for up/down arrow navigation)
    await saveMessage(query);
    resetNavigation();
    
    // 3️⃣ Run the query through the agent
    const result = await runQuery(query);
    
    // 4️⃣ Save agent's response to history
    if (result?.answer) {
      await updateAgentResponse(result.answer);
    }
  }, [exit, startSelection, runQuery, saveMessage, updateAgentResponse, resetNavigation]);
  
  // ... render UI
}
```

**What happens:**
1. User types query and presses Enter
2. Query is saved to input history
3. `runQuery()` is called (defined in useAgentRunner hook)
4. When complete, answer is saved to history

---

## Step 2: Agent Initialization (Hook Layer)

### File: `src/hooks/useAgentRunner.ts`

```typescript
export function useAgentRunner(agentConfig, inMemoryChatHistoryRef) {
  const runQuery = useCallback(async (query: string) => {
    // 1️⃣ Create AbortController for cancellation support
    const abortController = new AbortController();
    abortControllerRef.current = abortController;
    
    let finalAnswer: string | undefined;
    
    // 2️⃣ Add query to UI history immediately (shows as "processing")
    const itemId = Date.now().toString();
    const startTime = Date.now();
    setHistory(prev => [...prev, {
      id: itemId,
      query: "What's Apple's revenue for the last 4 quarters?",
      events: [],
      answer: '',
      status: 'processing',
      startTime,
    }]);
    
    // 3️⃣ Clear any previous errors
    setError(null);
    setWorkingState({ status: 'thinking' });
    
    try {
      // 4️⃣ Create agent instance with config and abort signal
      const agent = await Agent.create({
        model: 'gpt-5.2',
        modelProvider: 'openai',
        maxIterations: 10,
        signal: abortController.signal,
      });
      
      // 5️⃣ Run agent (returns async generator)
      const stream = agent.run(query, inMemoryChatHistoryRef.current!);
      
      // 6️⃣ Process events as they arrive
      for await (const event of stream) {
        if (event.type === 'done') {
          finalAnswer = (event as DoneEvent).answer;
        }
        handleEvent(event);  // Updates UI state
      }
      
      // 7️⃣ Return final answer
      if (finalAnswer) {
        return { answer: finalAnswer };
      }
    } catch (e) {
      // Handle errors...
    }
  }, [agentConfig, inMemoryChatHistoryRef, handleEvent]);
  
  return { runQuery, /* ... */ };
}
```

**What happens:**
1. Creates abort controller (allows cancellation with Esc/Ctrl+C)
2. Adds query to UI history immediately (shows it's processing)
3. Creates Agent instance with config
4. Runs agent and processes events as they stream in
5. Returns final answer when done

---

## Step 3: Agent Creation (Agent Core)

### File: `src/agent/agent.ts`

```typescript
export class Agent {
  static create(config: AgentConfig = {}): Agent {
    // 1️⃣ Get model from config (defaults to 'gpt-5.2')
    const model = config.model ?? 'gpt-5.2';
    
    // 2️⃣ Get all available tools for this model
    const tools = getTools(model);
    // Returns: [financial_search, web_search (if Tavily key present)]
    
    // 3️⃣ Build system prompt with tool descriptions
    const systemPrompt = buildSystemPrompt(model);
    // Injects detailed tool usage instructions
    
    // 4️⃣ Create agent instance
    return new Agent(config, tools, systemPrompt);
  }
  
  private constructor(config, tools, systemPrompt) {
    this.model = config.model ?? 'gpt-5.2';
    this.modelProvider = config.modelProvider ?? 'openai';
    this.maxIterations = config.maxIterations ?? 10;
    this.tools = tools;
    this.toolMap = new Map(tools.map(t => [t.name, t]));
    this.systemPrompt = systemPrompt;
    this.signal = config.signal;
  }
}
```

**What happens:**
1. Model defaults to 'gpt-5.2' if not specified
2. Tools are loaded (financial_search, optionally web_search)
3. System prompt is built with detailed tool descriptions
4. Agent instance is created with all configuration

---

## Step 4: Agent Loop - Iteration 1 (Planning)

### File: `src/agent/agent.ts`

```typescript
async *run(query: string, inMemoryHistory?: InMemoryChatHistory) {
  // 1️⃣ Create scratchpad for this query (single source of truth)
  const scratchpad = new Scratchpad(query);
  // Creates file: .dexter/scratchpad/20260122-153045_abc123def456.jsonl
  // Writes: {"type":"init","content":"What's Apple's...","timestamp":"..."}
  
  // 2️⃣ Build initial prompt with conversation history
  let currentPrompt = this.buildInitialPrompt(query, inMemoryHistory);
  // If first query: just the query
  // If follow-up: query + context from previous queries
  
  let iteration = 0;
  
  // 3️⃣ Main agent loop
  while (iteration < this.maxIterations) {
    iteration++; // Now at iteration 1
    
    // 4️⃣ Call LLM with current prompt
    const response = await this.callModel(currentPrompt);
    // Sends to OpenAI:
    //   System: "You are Dexter... [tool descriptions]..."
    //   User: "What's Apple's revenue for the last 4 quarters?"
    //   Tools: [financial_search, web_search]
    
    // 5️⃣ Extract text content from response
    const responseText = extractTextContent(response);
    // "I need to get Apple's income statements for the last 4 quarters."
    
    // 6️⃣ If there's thinking AND tool calls, emit thinking event
    if (responseText && hasToolCalls(response)) {
      scratchpad.addThinking(responseText);
      yield { type: 'thinking', message: responseText };
      // UI shows: 💭 "I need to get Apple's income statements..."
    }
    
    // 7️⃣ Check if LLM wants to use tools
    if (!hasToolCalls(response)) {
      // No tools needed - ready for final answer
      // (We'll come back to this in Step 6)
    }
    
    // 8️⃣ Execute tool calls
    const generator = this.executeToolCalls(response, query, scratchpad);
    let result = await generator.next();
    
    while (!result.done) {
      yield result.value;  // Emit tool events to UI
      result = await generator.next();
    }
    
    // 9️⃣ Build next prompt with tool summaries (context compaction!)
    currentPrompt = buildIterationPrompt(query, scratchpad.getToolSummaries());
    // Next prompt will have:
    //   "Query: What's Apple's revenue...
    //    Data retrieved: Retrieved 4 quarters of income statements for AAPL"
  }
}
```

**What happens:**
1. Scratchpad created (persists to disk)
2. Initial prompt built (query + any history)
3. LLM called with prompt and tools
4. LLM responds with thinking + tool calls
5. Thinking is emitted to UI
6. Tool execution begins (next step)

---

## Step 5: Tool Execution

### File: `src/agent/agent.ts`

```typescript
private async *executeToolCall(
  toolName: string,
  toolArgs: Record<string, unknown>,
  query: string,
  scratchpad: Scratchpad
) {
  // 1️⃣ Emit tool start event
  yield { 
    type: 'tool_start', 
    tool: 'financial_search', 
    args: { query: "Apple revenue last 4 quarters" }
  };
  // UI shows: 🔧 Using financial_search...
  
  const startTime = Date.now();
  
  try {
    // 2️⃣ Get tool from registry
    const tool = this.toolMap.get('financial_search');
    
    // 3️⃣ Invoke the tool
    const rawResult = await tool.invoke(
      { query: "Apple revenue last 4 quarters" },
      this.signal ? { signal: this.signal } : undefined
    );
    // Tool routes to: getIncomeStatements("AAPL", period: "quarterly", limit: 4)
    // Returns: JSON with 4 quarters of income statement data
    
    const result = typeof rawResult === 'string' 
      ? rawResult 
      : JSON.stringify(rawResult);
    
    const duration = Date.now() - startTime;
    
    // 4️⃣ Emit tool end event
    yield { 
      type: 'tool_end', 
      tool: 'financial_search', 
      args: { query: "..." },
      result: "{\"data\":[...]}",  // Full JSON
      duration: 1234
    };
    // UI shows: 🔧 financial_search(...) ✓ 1234ms
    
    // 5️⃣ Generate LLM summary for context compaction
    const llmSummary = await this.summarizeToolResult(
      query, 
      'financial_search', 
      { query: "..." }, 
      result
    );
    // Uses fast model (gpt-4.1) to generate:
    // "Retrieved 4 quarters of income statements for AAPL with revenue data"
    
    // 6️⃣ Add to scratchpad (full result + summary)
    scratchpad.addToolResult(
      'financial_search',
      { query: "..." },
      result,           // Full 10KB JSON
      llmSummary        // 100-byte summary
    );
    // Writes to file:
    // {"type":"tool_result","toolName":"financial_search",...}
    
  } catch (error) {
    // Handle errors (emit tool_error event)
  }
}
```

**What happens:**
1. Tool start event emitted (UI updates)
2. financial_search tool invoked
3. Tool routes query to getIncomeStatements API call
4. Full JSON result returned (10KB+)
5. Fast LLM generates 1-sentence summary
6. Both full result and summary saved to scratchpad

### File: `src/tools/finance/financial-search.ts`

```typescript
export const createFinancialSearch = (routerModel: string) => {
  return new DynamicStructuredTool({
    name: 'financial_search',
    description: 'Natural language interface to financial data APIs',
    schema: z.object({
      query: z.string().describe('Natural language financial query'),
    }),
    
    func: async ({ query }: FinancialSearchInput) => {
      // 1️⃣ Router LLM understands the query
      const routerPrompt = `Query: "${query}"\n\nWhich financial tool(s) should I use?`;
      const routerResponse = await callLlm(routerPrompt, {
        model: routerModel,  // gpt-5.2
        systemPrompt: ROUTER_SYSTEM_PROMPT,
        tools: FINANCE_TOOLS,  // All specialized tools
      });
      // Router decides: Use getIncomeStatements
      
      // 2️⃣ Execute selected tool
      const toolCalls = extractToolCalls(routerResponse);
      // toolCalls = [{ 
      //   name: 'getIncomeStatements',
      //   args: { ticker: 'AAPL', period: 'quarterly', limit: 4 }
      // }]
      
      const results = await Promise.all(
        toolCalls.map(async (call) => {
          const tool = FINANCE_TOOL_MAP.get(call.name);
          return await tool.invoke(call.args);
        })
      );
      
      // 3️⃣ Return formatted results
      return formatToolResult(results, query);
    }
  });
};
```

**What happens:**
1. financial_search receives: "Apple revenue last 4 quarters"
2. Internal router LLM analyzes query
3. Router decides to use getIncomeStatements tool
4. getIncomeStatements called with: ticker=AAPL, period=quarterly, limit=4
5. API returns 4 quarters of income statement data
6. Results formatted and returned to agent

---

## Step 6: Agent Loop - Iteration 2 (Deciding)

### Back to: `src/agent/agent.ts`

```typescript
async *run(query: string, inMemoryHistory?: InMemoryChatHistory) {
  // ... (iteration 1 completed)
  
  while (iteration < this.maxIterations) {
    iteration++; // Now at iteration 2
    
    // 1️⃣ Build prompt with tool SUMMARIES (not full data)
    currentPrompt = buildIterationPrompt(query, scratchpad.getToolSummaries());
    // Prompt now contains:
    //   "Query: What's Apple's revenue for the last 4 quarters?
    //    
    //    Data retrieved and work completed so far:
    //    - Retrieved 4 quarters of income statements for AAPL with revenue data
    //    
    //    Review the data above. If you have sufficient information to answer..."
    
    // 2️⃣ Call LLM again
    const response = await this.callModel(currentPrompt);
    // LLM thinks: "I have the data I need, ready to generate final answer"
    
    const responseText = extractTextContent(response);
    
    // 3️⃣ Check for tool calls
    if (!hasToolCalls(response)) {
      // ✅ No tool calls! Ready for final answer
      
      // 4️⃣ Check if this is a simple response (no tools were ever used)
      if (!scratchpad.hasToolResults() && responseText) {
        // Direct response (e.g., "Hello!" to greeting)
        yield { type: 'answer_start' };
        yield { type: 'answer_chunk', text: responseText };
        yield { type: 'done', answer: responseText, toolCalls: [], iterations: 2 };
        return;
      }
      
      // 5️⃣ Generate final answer with FULL context
      const answerGenerator = this.generateFinalAnswer(query, scratchpad);
      let fullAnswer = '';
      
      for await (const event of answerGenerator) {
        yield event;
        if (event.type === 'answer_chunk') {
          fullAnswer += event.text;
        }
      }
      
      // 6️⃣ Done!
      yield { 
        type: 'done', 
        answer: fullAnswer,
        toolCalls: scratchpad.getToolCallRecords(),
        iterations: 2
      };
      return;
    }
    
    // If there were tool calls, loop would continue...
  }
}
```

**What happens:**
1. New prompt built with tool SUMMARY (not full 10KB data)
2. LLM sees summary and decides it has enough info
3. No tool calls in response - ready for final answer
4. Final answer generation begins (next step)

---

## Step 7: Final Answer Generation

### File: `src/agent/agent.ts`

```typescript
private async *generateFinalAnswer(
  query: string,
  scratchpad: Scratchpad
) {
  // 1️⃣ Emit answer start event
  yield { type: 'answer_start' };
  // UI shows: ✍️ Writing response...
  
  // 2️⃣ Load FULL context from scratchpad (not summaries!)
  const fullContext = this.buildFullContextForAnswer(scratchpad);
  // Returns:
  // "### financial_search(query='Apple revenue last 4 quarters')
  //  ```json
  //  {
  //    "data": [
  //      { "period": "Q4-2024", "revenue": 94900000000, ... },
  //      { "period": "Q3-2024", "revenue": 85800000000, ... },
  //      { "period": "Q2-2024", "revenue": 90800000000, ... },
  //      { "period": "Q1-2024", "revenue": 119600000000, ... }
  //    ]
  //  }
  //  ```"
  
  // 3️⃣ Build final answer prompt with complete data
  const prompt = buildFinalAnswerPrompt(query, fullContext);
  // "Query: What's Apple's revenue for the last 4 quarters?
  //  
  //  Data:
  //  [full JSON above]
  //  
  //  Answer proportionally - match depth to the question's complexity."
  
  // 4️⃣ Stream the answer
  const stream = streamLlmResponse(prompt, {
    model: this.model,
    systemPrompt: this.systemPrompt,
  });
  
  // 5️⃣ Yield chunks as they arrive
  for await (const chunk of stream) {
    yield { type: 'answer_chunk', text: chunk };
    // First chunk: "Apple's "
    // Second chunk: "revenue "
    // Third chunk: "for "
    // ... etc
  }
  // UI displays text as it streams in, feels instant!
}
```

**What happens:**
1. Answer start event emitted (UI updates)
2. FULL tool results loaded from scratchpad
3. Final answer prompt built with complete data
4. LLM generates answer, streaming chunks
5. Each chunk emitted to UI (text appears in real-time)

---

## Step 8: UI Updates (Event Processing)

### File: `src/hooks/useAgentRunner.ts`

```typescript
const handleEvent = useCallback((event: AgentEvent) => {
  switch (event.type) {
    // Iteration 1 - Thinking
    case 'thinking':
      setWorkingState({ status: 'thinking' });
      updateLastHistoryItem(item => ({
        events: [...item.events, {
          id: `thinking-${Date.now()}`,
          event,
          completed: true,
        }],
      }));
      break;
    
    // Iteration 1 - Tool Start
    case 'tool_start': {
      const toolId = `tool-${event.tool}-${Date.now()}`;
      setWorkingState({ status: 'tool', toolName: event.tool });
      updateLastHistoryItem(item => ({
        activeToolId: toolId,
        events: [...item.events, {
          id: toolId,
          event,  // { type: 'tool_start', tool: 'financial_search', ... }
          completed: false,
        }],
      }));
      break;
    }
    
    // Iteration 1 - Tool End
    case 'tool_end':
      setWorkingState({ status: 'thinking' });
      updateLastHistoryItem(item => ({
        activeToolId: undefined,
        events: item.events.map(e => 
          e.id === item.activeToolId
            ? { ...e, completed: true, endEvent: event }
            : e
        ),
      }));
      break;
    
    // Iteration 2 - Answer Start
    case 'answer_start':
      setWorkingState({ status: 'answering' });
      break;
    
    // Iteration 2 - Answer Chunks (streaming)
    case 'answer_chunk':
      // Hide "Writing response..." indicator
      setWorkingState({ status: 'idle' });
      // Append text to answer
      updateLastHistoryItem(item => ({
        answer: item.answer + event.text,
      }));
      // UI immediately shows: "Apple's revenue for the last 4 quarters..."
      break;
    
    // Iteration 2 - Done
    case 'done': {
      const doneEvent = event as DoneEvent;
      updateLastHistoryItem(item => {
        // Add to conversation history for multi-turn context
        if (item.query && doneEvent.answer) {
          inMemoryChatHistoryRef.current?.addMessage(
            item.query, 
            doneEvent.answer
          );
        }
        return {
          answer: doneEvent.answer,
          status: 'complete',
          duration: item.startTime ? Date.now() - item.startTime : undefined,
        };
      });
      setWorkingState({ status: 'idle' });
      break;
    }
  }
}, [updateLastHistoryItem, inMemoryChatHistoryRef]);
```

**What happens:**
1. Each event triggers UI state update
2. Thinking event → shows "💭 Thinking..."
3. Tool start → shows "🔧 Using financial_search..."
4. Tool end → shows "🔧 financial_search(...) ✓ 1234ms"
5. Answer chunks → text appears in real-time
6. Done → marks query as complete, saves to conversation history

---

## Step 9: Display in Terminal

### File: `src/cli.tsx`

```typescript
export function CLI() {
  return (
    <Box flexDirection="column">
      <Intro provider={provider} model={model} />
      
      {/* All history items (queries, events, answers) */}
      {history.map(item => (
        <HistoryItemView key={item.id} item={item} />
      ))}
      
      {/* Working indicator */}
      {isProcessing && <WorkingIndicator state={workingState} />}
      
      {/* Input */}
      <Box marginTop={1}>
        <Input onSubmit={handleSubmit} />
      </Box>
    </Box>
  );
}
```

### File: `src/components/HistoryItemView.tsx`

```typescript
export function HistoryItemView({ item }: { item: HistoryItem }) {
  return (
    <Box flexDirection="column" marginBottom={1}>
      {/* User query */}
      <Text color="cyan">➤ What's Apple's revenue for the last 4 quarters?</Text>
      
      {/* Tool executions */}
      {item.events.map(event => (
        <AgentEventView key={event.id} event={event} />
      ))}
      // Shows:
      // 💭 I need to get Apple's income statements for the last 4 quarters.
      // 🔧 financial_search(query="Apple revenue last 4 quarters") ✓ 1234ms
      
      {/* Answer */}
      {item.answer && (
        <Box marginTop={1}>
          <Text>
            Apple's revenue for the last 4 quarters:
            
            Q4 2024: $94.9B
            Q3 2024: $85.8B
            Q2 2024: $90.8B
            Q1 2024: $119.6B
            
            Total annual revenue: $391.1B
          </Text>
        </Box>
      )}
      
      {/* Status */}
      {item.status === 'complete' && item.duration && (
        <Text color="gray" dimColor>
          Completed in {item.duration}ms
        </Text>
      )}
    </Box>
  );
}
```

**What's displayed in terminal:**

```
Dexter 🤖  (openai/gpt-5.2)

➤ What's Apple's revenue for the last 4 quarters?
💭 I need to get Apple's income statements for the last 4 quarters.
🔧 financial_search(query="Apple revenue last 4 quarters") ✓ 1234ms

Apple's revenue for the last 4 quarters:

Q4 2024: $94.9B
Q3 2024: $85.8B
Q2 2024: $90.8B
Q1 2024: $119.6B

Total annual revenue: $391.1B

Completed in 5678ms

➤ _
```

---

## Complete Data Flow Summary

```
User Input
   ↓
CLI Component (handleSubmit)
   ↓
useAgentRunner Hook (runQuery)
   ↓
Agent.create()
   ├─ Load tools (financial_search, web_search)
   ├─ Build system prompt
   └─ Create agent instance
   ↓
Agent.run() - Iteration 1
   ├─ Create scratchpad
   ├─ Call LLM with query
   ├─ LLM responds with: thinking + tool_calls
   ├─ Yield thinking event → UI shows "💭 Thinking..."
   ├─ Execute tool: financial_search
   │  ├─ Yield tool_start → UI shows "🔧 Using financial_search..."
   │  ├─ financial_search routes to getIncomeStatements
   │  ├─ API call returns JSON data (10KB)
   │  ├─ Yield tool_end → UI shows "✓ 1234ms"
   │  ├─ Generate summary with fast LLM (100 bytes)
   │  └─ Save to scratchpad: full result + summary
   └─ Build next prompt with SUMMARY
   ↓
Agent.run() - Iteration 2
   ├─ Call LLM with summary
   ├─ LLM decides: "I have enough data"
   ├─ No tool calls in response
   ├─ Load FULL context from scratchpad
   ├─ Call LLM for final answer with full data
   ├─ Yield answer_start → UI shows "✍️ Writing..."
   ├─ Stream answer chunks
   │  ├─ Yield "Apple's " → UI displays
   │  ├─ Yield "revenue " → UI appends
   │  └─ ... continue streaming
   ├─ Yield done event
   └─ Save to conversation history
   ↓
UI Updates
   ├─ Display complete answer
   ├─ Mark as "complete"
   └─ Ready for next query
```

---

## Key Code Patterns

### 1. Event-Driven Architecture

```typescript
// Generator pattern for streaming events
async *run(query: string): AsyncGenerator<AgentEvent> {
  yield { type: 'thinking', message: '...' };
  yield { type: 'tool_start', tool: '...' };
  yield { type: 'tool_end', result: '...' };
  yield { type: 'answer_chunk', text: '...' };
  yield { type: 'done', answer: '...' };
}
```

### 2. Context Compaction

```typescript
// During loop: Use summaries (small)
const prompt = buildIterationPrompt(query, scratchpad.getToolSummaries());

// For final answer: Use full data (complete)
const fullContext = this.buildFullContextForAnswer(scratchpad);
const prompt = buildFinalAnswerPrompt(query, fullContext);
```

### 3. Abort Signal Propagation

```typescript
// Top level
const abortController = new AbortController();

// Passed through entire stack
const agent = Agent.create({ signal: abortController.signal });

// Used in tool execution
await tool.invoke(args, { signal: this.signal });

// User cancels
abortController.abort(); // Everything stops gracefully
```

### 4. React State Management

```typescript
// Immutable state updates
setHistory(prev => [...prev, newItem]);
setHistory(prev => [...prev.slice(0, -1), { ...prev[prev.length - 1], ...updates }]);
```

### 5. Error Handling

```typescript
try {
  const result = await tool.invoke(args);
  yield { type: 'tool_end', result };
} catch (error) {
  yield { type: 'tool_error', error: error.message };
  // Error is added to scratchpad - agent can adapt
}
```

---

## Performance Optimizations Explained

### 1. Fast Models for Summaries

```typescript
// Summarization doesn't need GPT-5.2
const summary = await callLlm(prompt, {
  model: getFastModel(this.modelProvider, this.model),  // Uses gpt-4.1
});
// 5-10x faster, much cheaper
```

### 2. Context Window Savings

```
Without compaction:
  Iteration 1: 10KB result
  Iteration 2: 10KB + 10KB = 20KB
  Iteration 3: 10KB + 10KB + 10KB = 30KB
  Total context: 60KB

With compaction:
  Iteration 1: 100 byte summary
  Iteration 2: 100 + 100 = 200 bytes
  Iteration 3: 100 + 100 + 100 = 300 bytes
  Final answer: 30KB full data
  Total context: 30.6KB (50% reduction)
```

### 3. Streaming UX

```typescript
// Without streaming: Wait 5 seconds, show complete answer
await getFullAnswer(); // user sees nothing for 5s
displayAnswer(fullAnswer);

// With streaming: Text appears immediately
for await (const chunk of streamAnswer()) {
  displayChunk(chunk); // user sees text appearing instantly
}
// Feels 10x faster even though total time is the same!
```

---

## Scratchpad File Example

After the query completes, `.dexter/scratchpad/20260122-153045_abc123def456.jsonl`:

```jsonl
{"type":"init","content":"What's Apple's revenue for the last 4 quarters?","timestamp":"2026-01-22T15:30:45.123Z"}
{"type":"thinking","content":"I need to get Apple's income statements for the last 4 quarters.","timestamp":"2026-01-22T15:30:46.234Z"}
{"type":"tool_result","toolName":"financial_search","args":{"query":"Apple revenue last 4 quarters"},"result":{"data":[{"period":"Q4-2024","revenue":94900000000,...},{"period":"Q3-2024","revenue":85800000000,...},{"period":"Q2-2024","revenue":90800000000,...},{"period":"Q1-2024","revenue":119600000000,...}]},"llmSummary":"Retrieved 4 quarters of income statements for AAPL with revenue data","timestamp":"2026-01-22T15:30:47.456Z"}
```

Each line is a complete JSON object. This format is:
- **Resilient**: Corruption of one line doesn't break others
- **Appendable**: Easy to add new lines without reading entire file
- **Parseable**: Can parse line-by-line for large files

---

## Conclusion

This walkthrough demonstrated how a simple query flows through the entire Dexter system:

1. **User Input** → CLI handles submission
2. **Hook Layer** → useAgentRunner orchestrates execution
3. **Agent Core** → ReAct loop with tool execution
4. **Tool System** → Routes and executes financial data queries
5. **Context Compaction** → Summaries during loop, full data at end
6. **Streaming** → Real-time UI updates
7. **Final Display** → Complete answer shown in terminal

The architecture is elegant, efficient, and extensible - a great example of modern AI agent design!
