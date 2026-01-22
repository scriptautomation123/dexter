# Dexter Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Core Components](#core-components)
4. [Agent Loop Execution Flow](#agent-loop-execution-flow)
5. [Key Code Sections Explained](#key-code-sections-explained)
6. [Data Flow](#data-flow)
7. [Tool System](#tool-system)
8. [Multi-Turn Conversations](#multi-turn-conversations)
9. [UI Components](#ui-components)

## Overview

**Dexter** is an autonomous AI agent built for financial research. It uses a **ReAct (Reasoning + Acting)** pattern where the AI:
1. Thinks about what data it needs
2. Executes tools to gather that data
3. Reflects on the results
4. Iterates until it can answer the user's query

The project is built with:
- **Runtime**: Bun (fast JavaScript runtime)
- **Language**: TypeScript
- **UI Framework**: Ink (React for CLIs)
- **AI Framework**: LangChain
- **LLM Providers**: OpenAI, Anthropic, Google, xAI, Ollama

The codebase has ~3,600 lines of TypeScript code organized into a clean, modular architecture.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        User Input                           │
│                      (Terminal CLI)                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                      CLI Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Input      │  │    Intro     │  │  Model       │     │
│  │  Component   │  │  Component   │  │  Selector    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   Agent Runner Hook                         │
│           (useAgentRunner - Orchestration)                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent Core                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Agent      │  │  Scratchpad  │  │   Prompts    │     │
│  │   Loop       │  │  (Memory)    │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────┬────────────────────────────────────┬───────────────┘
         │                                    │
         ▼                                    ▼
┌─────────────────────┐           ┌──────────────────────────┐
│    LLM Provider     │           │     Tool Registry        │
│  ┌──────────────┐   │           │  ┌─────────────────┐    │
│  │   OpenAI     │   │           │  │ Financial Tools │    │
│  │  Anthropic   │   │           │  │   - Prices      │    │
│  │   Google     │   │           │  │   - Statements  │    │
│  │   Ollama     │   │           │  │   - Metrics     │    │
│  └──────────────┘   │           │  │   - News        │    │
└─────────────────────┘           │  └─────────────────┘    │
                                  │  ┌─────────────────┐    │
                                  │  │  Web Search     │    │
                                  │  │  (Tavily)       │    │
                                  │  └─────────────────┘    │
                                  └──────────────────────────┘
```

## Core Components

### 1. Entry Point (`src/index.tsx`)

The application entry point is simple and elegant:

```typescript
#!/usr/bin/env bun
import React from 'react';
import { render } from 'ink';
import { config } from 'dotenv';
import { CLI } from './cli.js';

// Load environment variables
config({ quiet: true });

// Render the CLI app
render(<CLI />);
```

**What it does:**
- Loads environment variables (API keys)
- Renders the React-based CLI using Ink
- The `#!/usr/bin/env bun` shebang makes it executable

### 2. CLI Component (`src/cli.tsx`)

This is the main UI orchestrator - a 210-line React component that:

**Key Responsibilities:**
- Manages UI state (model selection, agent execution)
- Handles user input and keyboard shortcuts
- Displays conversation history
- Shows real-time progress (thinking, tool calls, answers)

**Important Code Section:**

```typescript
const handleSubmit = useCallback(async (query: string) => {
  // Handle exit
  if (query.toLowerCase() === 'exit' || query.toLowerCase() === 'quit') {
    console.log('Goodbye!');
    exit();
    return;
  }
  
  // Handle model selection command
  if (query === '/model') {
    startSelection();
    return;
  }
  
  // Run query and save agent response when complete
  const result = await runQuery(query);
  if (result?.answer) {
    await updateAgentResponse(result.answer);
  }
}, [exit, startSelection, isInSelectionFlow, workingState.status, runQuery, saveMessage, updateAgentResponse, resetNavigation]);
```

This function routes special commands (`/model`, `exit`) and normal queries to the agent.

### 3. Agent Core (`src/agent/agent.ts`)

**This is the heart of Dexter** - a 312-line class that implements the ReAct loop.

#### Key Architecture Decisions:

**Event-Driven Streaming:**
The agent uses `async generators` to stream events in real-time:

```typescript
async *run(query: string, inMemoryHistory?: InMemoryChatHistory): AsyncGenerator<AgentEvent>
```

Events include:
- `thinking` - Agent is reasoning
- `tool_start` - Tool execution begins
- `tool_end` - Tool execution completes
- `answer_chunk` - Streaming final answer
- `done` - Complete with results

**The Main Agent Loop:**

```typescript
// Main agent loop
while (iteration < this.maxIterations) {
  iteration++;

  const response = await this.callModel(currentPrompt);
  const responseText = extractTextContent(response);

  // Emit thinking if there are also tool calls
  if (responseText && hasToolCalls(response)) {
    scratchpad.addThinking(responseText);
    yield { type: 'thinking', message: responseText };
  }

  // No tool calls = ready to generate final answer
  if (!hasToolCalls(response)) {
    // If no tools were called at all, just use the direct response
    if (!scratchpad.hasToolResults() && responseText) {
      yield { type: 'answer_start' };
      yield { type: 'answer_chunk', text: responseText };
      yield { type: 'done', answer: responseText, toolCalls: [], iterations: iteration };
      return;
    }

    // Stream final answer with full context from scratchpad
    const answerGenerator = this.generateFinalAnswer(query, scratchpad);
    // ... stream the answer
    return;
  }

  // Execute tools and add results to scratchpad
  const generator = this.executeToolCalls(response, query, scratchpad);
  // ... execute tools
  
  // Build iteration prompt from scratchpad
  currentPrompt = buildIterationPrompt(query, scratchpad.getToolSummaries());
}
```

**How it works:**
1. **Call LLM** with the current prompt
2. **Check for tool calls** in the response
3. **If no tool calls**: Generate the final answer and exit
4. **If tool calls**: Execute them and continue to next iteration
5. **Build next prompt** with tool results (as summaries)
6. **Repeat** until answer is ready or max iterations reached

### 4. Scratchpad (`src/agent/scratchpad.ts`)

The scratchpad is the **single source of truth** for all agent work. It's an append-only log that persists to disk.

**Key Innovation - Context Compaction:**

The scratchpad stores both:
- **Full tool results** (complete JSON data)
- **LLM-generated summaries** (concise 1-sentence summaries)

During the agent loop, only summaries are used to save context window space. When generating the final answer, full data is loaded:

```typescript
addToolResult(
  toolName: string,
  args: Record<string, unknown>,
  result: string,
  llmSummary: string
): void {
  this.append({
    type: 'tool_result',
    timestamp: new Date().toISOString(),
    toolName,
    args,
    result: this.parseResultSafely(result),
    llmSummary,
  });
}

// Get summaries for iteration prompts (lightweight)
getToolSummaries(): string[] {
  return this.readEntries()
    .filter(e => e.type === 'tool_result' && e.llmSummary)
    .map(e => e.llmSummary!);
}

// Get full contexts for final answer (complete data)
getFullContexts(): ToolContext[] {
  return this.readEntries()
    .filter(e => e.type === 'tool_result' && e.toolName && e.result)
    .map(e => ({
      toolName: e.toolName!,
      args: e.args!,
      result: this.stringifyResult(e.result),
    }));
}
```

**File Format:** JSONL (newline-delimited JSON) for resilient appending
**Storage:** `.dexter/scratchpad/TIMESTAMP_HASH.jsonl`

This design allows for:
- Debug/audit trail of agent reasoning
- Recovery from crashes
- Efficient context window management

### 5. Tool System (`src/tools/`)

Dexter has a sophisticated tool system with two main categories:

#### Financial Tools (`src/tools/finance/`)

The `financial_search` tool is a **meta-tool** - it routes natural language queries to the appropriate specialized tools:

```typescript
const FINANCE_TOOLS: StructuredToolInterface[] = [
  // Price Data
  getPriceSnapshot,
  getPrices,
  getCryptoPriceSnapshot,
  getCryptoPrices,
  // Fundamentals
  getIncomeStatements,
  getBalanceSheets,
  getCashFlowStatements,
  // Metrics & Estimates
  getFinancialMetricsSnapshot,
  getAnalystEstimates,
  // SEC Filings
  getFilings,
  get10KFilingItems,
  // Other
  getNews,
  getInsiderTrades,
  getSegmentedRevenues,
];
```

**How it works:**

1. User asks: "What's Apple's revenue growth?"
2. Agent calls: `financial_search("What's Apple's revenue growth?")`
3. Financial search tool:
   - Uses LLM to understand the query
   - Routes to `getIncomeStatements` tool
   - Executes and returns results
4. Agent receives structured financial data

This routing layer simplifies the agent's decision-making - it only needs to choose between `financial_search` and `web_search`.

#### Web Search Tool (`src/tools/search/`)

Uses Tavily API for general web searches when financial data isn't sufficient.

### 6. LLM Abstraction (`src/model/llm.ts`)

Dexter supports multiple LLM providers through a unified interface:

```typescript
const MODEL_PROVIDERS: Record<string, ModelFactory> = {
  'claude-': (name, opts) => new ChatAnthropic({ model: name, ...opts }),
  'gemini-': (name, opts) => new ChatGoogleGenerativeAI({ model: name, ...opts }),
  'grok-': (name, opts) => new ChatOpenAI({ 
    model: name, 
    configuration: { baseURL: 'https://api.x.ai/v1' },
    ...opts 
  }),
  'gpt-': (name, opts) => new ChatOpenAI({ model: name, ...opts }),
};
```

The system auto-detects the provider by model name prefix. It also has a fast model concept for lightweight tasks:

```typescript
const FAST_MODELS: Record<string, string> = {
  openai: 'gpt-4.1',
  anthropic: 'claude-haiku-4-5',
  google: 'gemini-3-flash-preview',
  xai: 'grok-4-1-fast-reasoning',
};
```

Fast models are used for:
- Tool result summarization
- Message summarization in multi-turn conversations
- Any task that doesn't require deep reasoning

### 7. Agent Runner Hook (`src/hooks/useAgentRunner.ts`)

This React hook bridges the UI and the agent core:

```typescript
export function useAgentRunner(
  agentConfig: AgentConfig,
  inMemoryChatHistoryRef: React.RefObject<InMemoryChatHistory>
): UseAgentRunnerResult {
  const [history, setHistory] = useState<HistoryItem[]>([]);
  const [workingState, setWorkingState] = useState<WorkingState>({ status: 'idle' });
  const [error, setError] = useState<string | null>(null);
  
  const abortControllerRef = useRef<AbortController | null>(null);
```

**Key Features:**
- Manages conversation history (all queries and answers)
- Tracks working state (idle, thinking, using tool, answering)
- Handles cancellation via AbortController
- Processes agent events and updates UI state

**Event Handling:**

```typescript
const handleEvent = useCallback((event: AgentEvent) => {
  switch (event.type) {
    case 'thinking':
      setWorkingState({ status: 'thinking' });
      updateLastHistoryItem(item => ({
        events: [...item.events, { id: `thinking-${Date.now()}`, event, completed: true }],
      }));
      break;
      
    case 'tool_start':
      setWorkingState({ status: 'tool', toolName: event.tool });
      updateLastHistoryItem(item => ({
        activeToolId: toolId,
        events: [...item.events, { id: toolId, event, completed: false }],
      }));
      break;
      
    case 'answer_chunk':
      updateLastHistoryItem(item => ({
        answer: item.answer + event.text,
      }));
      break;
  }
}, [updateLastHistoryItem, inMemoryChatHistoryRef]);
```

## Agent Loop Execution Flow

Here's a detailed walkthrough of what happens when a user asks: **"What's Apple's revenue for the last 4 quarters?"**

### Step 1: Query Submission
```
User Input: "What's Apple's revenue for the last 4 quarters?"
       ↓
CLI Component (handleSubmit)
       ↓
useAgentRunner.runQuery()
       ↓
Creates new Agent instance
       ↓
agent.run(query)
```

### Step 2: Iteration 1 - Planning

```
Agent builds prompt with:
- System prompt (with tool descriptions)
- User query
- Conversation history (if any)

       ↓
Calls LLM
       ↓
LLM Response:
  Content: "I need to get Apple's income statements for the last 4 quarters."
  Tool Calls: [
    {
      name: "financial_search",
      args: { query: "Apple revenue last 4 quarters" }
    }
  ]
       ↓
Agent yields: { type: 'thinking', message: "I need to..." }
       ↓
Agent executes tool: financial_search
       ↓
Tool routes to: getIncomeStatements("AAPL", period: "quarterly", limit: 4)
       ↓
API Call to Financial Datasets API
       ↓
Returns: [Q4 2024, Q3 2024, Q2 2024, Q1 2024 income statements]
       ↓
Agent yields: { type: 'tool_end', result: "..." }
       ↓
LLM summarizes: "Retrieved 4 quarters of income statements for AAPL"
       ↓
Scratchpad saves:
  - Full result (complete JSON)
  - Summary (one sentence)
```

### Step 3: Iteration 2 - Deciding to Answer

```
Agent builds new prompt with:
- Original query
- Tool summary: "Retrieved 4 quarters of income statements for AAPL"

       ↓
Calls LLM
       ↓
LLM Response:
  Content: "I have the data, ready to answer."
  Tool Calls: [] (empty - no more tools needed)
       ↓
Agent detects: No tool calls = ready for final answer
       ↓
Agent loads FULL context from scratchpad
       ↓
Builds final answer prompt with complete JSON data
       ↓
Streams final answer
```

### Step 4: Final Answer Generation

```
Agent yields: { type: 'answer_start' }
       ↓
Streams chunks:
  { type: 'answer_chunk', text: "Apple's " }
  { type: 'answer_chunk', text: "revenue " }
  { type: 'answer_chunk', text: "for the " }
  ...
       ↓
Complete answer displayed in terminal:
"Apple's revenue for the last 4 quarters:
Q4 2024: $94.9B
Q3 2024: $85.8B
Q2 2024: $90.8B
Q1 2024: $119.6B"
       ↓
Agent yields: { type: 'done', answer: "...", iterations: 2 }
       ↓
useAgentRunner updates history
       ↓
UI displays complete answer
```

### Complete Flow Diagram

```
┌──────────────┐
│  User Query  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│  Iteration 1: Planning & Tool Selection          │
│  ┌────────────────────────────────────────────┐  │
│  │ LLM Call with Query + System Prompt        │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ Response: Thinking + Tool Calls            │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ Execute Tools (e.g., financial_search)     │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ Save to Scratchpad:                        │  │
│  │  - Full Result (complete data)             │  │
│  │  - LLM Summary (1 sentence)                │  │
│  └────────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  Iteration 2+: Review & Decide                   │
│  ┌────────────────────────────────────────────┐  │
│  │ LLM Call with Query + Tool Summaries       │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ Decision Point:                            │  │
│  │  • More tools needed? → Execute & repeat   │  │
│  │  • Have enough data? → Generate answer     │  │
│  └────────────┬───────────────────────────────┘  │
└───────────────┼───────────────────────────────────┘
                │
                ▼ (Enough data)
┌──────────────────────────────────────────────────┐
│  Final Answer Generation                         │
│  ┌────────────────────────────────────────────┐  │
│  │ Load FULL context from scratchpad         │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ LLM Call with Query + Full Data            │  │
│  └────────────┬───────────────────────────────┘  │
│               ▼                                   │
│  ┌────────────────────────────────────────────┐  │
│  │ Stream Answer Chunks to UI                 │  │
│  └────────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────┐
│  Display Final Answer    │
└──────────────────────────┘
```

## Data Flow

### Context Window Management

One of Dexter's clever optimizations is **context compaction**:

**Problem:** Tool results can be large (10KB+ JSON). With multiple tools, context window fills up quickly.

**Solution:** Two-phase approach:

1. **During Agent Loop (Iterations):**
   - Store full results in scratchpad (on disk)
   - Generate LLM summaries (1 sentence each)
   - Use only summaries in iteration prompts

2. **During Final Answer:**
   - Load full results from scratchpad
   - Provide complete data to LLM for accurate answer

**Code Implementation:**

```typescript
// After tool execution
private async summarizeToolResult(
  query: string,
  toolName: string,
  toolArgs: Record<string, unknown>,
  result: string
): Promise<string> {
  const prompt = buildToolSummaryPrompt(query, toolName, toolArgs, result);
  const summary = await callLlm(prompt, {
    model: getFastModel(this.modelProvider, this.model), // Use fast model
    systemPrompt: 'You are a concise data summarizer.',
  });
  return String(summary);
}

// Build iteration prompt (uses summaries)
currentPrompt = buildIterationPrompt(query, scratchpad.getToolSummaries());

// Build final answer prompt (uses full data)
const fullContext = this.buildFullContextForAnswer(scratchpad);
const prompt = buildFinalAnswerPrompt(query, fullContext);
```

### Message Flow

```
User Types → Input Component
                ↓
         CLI handleSubmit
                ↓
      useAgentRunner.runQuery()
                ↓
         Agent.run() generator
                ↓
    Yields AgentEvent stream:
      - ThinkingEvent
      - ToolStartEvent
      - ToolEndEvent
      - AnswerChunkEvent
      - DoneEvent
                ↓
    useAgentRunner.handleEvent()
                ↓
     Updates React state:
       - history
       - workingState
       - error
                ↓
    CLI re-renders with new state
                ↓
      UI Components display:
        - HistoryItemView (past queries/answers)
        - WorkingIndicator (current status)
        - Input (next query)
```

## Tool System

### Tool Registry Architecture

The tool registry (`src/tools/registry.ts`) provides a centralized way to manage tools:

```typescript
export interface RegisteredTool {
  name: string;                           // Tool identifier
  tool: StructuredToolInterface;           // Actual LangChain tool
  description: string;                     // Rich description for prompts
}

export function getToolRegistry(model: string): RegisteredTool[] {
  const tools: RegisteredTool[] = [
    {
      name: 'financial_search',
      tool: createFinancialSearch(model),
      description: FINANCIAL_SEARCH_DESCRIPTION,
    },
  ];

  // Conditionally add web_search if API key is present
  if (process.env.TAVILY_API_KEY) {
    tools.push({
      name: 'web_search',
      tool: tavilySearch,
      description: WEB_SEARCH_DESCRIPTION,
    });
  }

  return tools;
}
```

### Tool Descriptions

Tool descriptions are detailed markdown documents that guide the LLM:

**Example from `src/tools/descriptions/financial-search.ts`:**

```typescript
export const FINANCIAL_SEARCH_DESCRIPTION = `
A natural language interface to financial data APIs.

**When to use:**
- ANY query about stock prices, financial metrics, company fundamentals
- Earnings data, revenue, profit margins, cash flow
- Company news, analyst estimates, insider trades

**When NOT to use:**
- General knowledge questions
- Non-financial web searches (use web_search instead)

**How it works:**
Pass your full natural language query. The tool automatically:
1. Understands what data you need
2. Routes to the appropriate financial API
3. Returns structured results

**Examples:**
- "Apple's revenue growth over the last 5 years"
- "Compare P/E ratios of Tesla and Ford"
- "Latest news about Microsoft"
`;
```

These descriptions are injected into the system prompt, teaching the LLM when and how to use each tool.

### Financial Search Implementation

The financial search tool is a **router** that uses an LLM to understand queries and select the right tool:

```typescript
async _call(input: FinancialSearchInput): Promise<string> {
  const query = input.query;
  
  // Step 1: Router LLM decides which tool(s) to use
  const routerPrompt = `Query: ${query}\n\nWhich financial tool(s) should I use?`;
  const routerResponse = await callLlm(routerPrompt, {
    model: this.routerModel,
    systemPrompt: ROUTER_SYSTEM_PROMPT,
    tools: FINANCE_TOOLS,
  });

  // Step 2: Execute selected tools
  const toolCalls = extractToolCalls(routerResponse);
  const results = await Promise.all(
    toolCalls.map(async (call) => {
      const tool = FINANCE_TOOL_MAP.get(call.name);
      return await tool.invoke(call.args);
    })
  );

  // Step 3: Combine and format results
  return formatResults(results);
}
```

This architecture allows adding new financial data sources without modifying the agent - just add new tools to the registry.

## Multi-Turn Conversations

Dexter maintains conversation context across queries using `InMemoryChatHistory` (`src/utils/in-memory-chat-history.ts`).

### Key Features

1. **Automatic Summarization:** Each answer is summarized by an LLM for efficient context
2. **Relevance Filtering:** Only relevant previous messages are included in context
3. **Caching:** Relevance decisions are cached per query to avoid redundant LLM calls

### How It Works

```typescript
export class InMemoryChatHistory {
  private messages: Message[] = [];
  private relevantMessagesByQuery: Map<string, Message[]> = new Map();

  // Add a new conversation turn
  async addMessage(query: string, answer: string): Promise<void> {
    const summary = await this.generateSummary(query, answer);
    this.messages.push({
      id: this.messages.length,
      query,
      answer,
      summary,  // LLM-generated 1-2 sentence summary
    });
  }

  // Select relevant messages for current query
  async selectRelevantMessages(currentQuery: string): Promise<Message[]> {
    // Check cache first
    const cached = this.relevantMessagesByQuery.get(this.hashQuery(currentQuery));
    if (cached) return cached;

    // Use LLM to determine relevance
    const prompt = `Current query: "${currentQuery}"
    
Previous conversations:
${JSON.stringify(this.messages.map(m => ({
  id: m.id,
  query: m.query,
  summary: m.summary
})), null, 2)}

Select which previous messages are relevant.`;

    const response = await callLlm(prompt, {
      systemPrompt: MESSAGE_SELECTION_SYSTEM_PROMPT,
      outputSchema: SelectedMessagesSchema,  // Structured output
    });

    const selectedIds = response.message_ids;
    const selected = selectedIds.map(id => this.messages[id]);
    
    // Cache the result
    this.relevantMessagesByQuery.set(this.hashQuery(currentQuery), selected);
    return selected;
  }
}
```

### Example Conversation Flow

```
User: "What's Apple's revenue?"
Bot: "Apple's revenue for Q4 2024 is $94.9B"
  └─> Stored in history with summary: "Provided Apple Q4 2024 revenue"

User: "How does that compare to last year?"
  └─> System detects "that" refers to previous context
  └─> Loads previous message: "What's Apple's revenue?"
  └─> Agent understands to compare Q4 2024 vs Q4 2023
  └─> Calls tools with proper context

Bot: "Apple's Q4 2023 revenue was $89.5B. Growth of 6%."
```

## UI Components

Dexter uses **Ink** (React for terminals) for its UI. Key components:

### 1. Input Component (`src/components/Input.tsx`)

Handles user input with history navigation:

```typescript
export function Input({ onSubmit, historyValue, onHistoryNavigate }: InputProps) {
  useInput((input, key) => {
    if (key.upArrow) {
      onHistoryNavigate('up');  // Navigate to previous query
    } else if (key.downArrow) {
      onHistoryNavigate('down');  // Navigate to next query
    }
  });

  return (
    <Box>
      <Text color="cyan">➤ </Text>
      <TextInput
        value={inputValue}
        onChange={setInputValue}
        onSubmit={handleSubmit}
      />
    </Box>
  );
}
```

Features:
- Arrow key navigation through query history
- Ctrl+C to cancel
- `/model` command to switch models
- `exit` or `quit` to exit

### 2. WorkingIndicator Component

Shows real-time status:

```typescript
export function WorkingIndicator({ state }: { state: WorkingState }) {
  if (state.status === 'idle') return null;

  return (
    <Box marginBottom={1}>
      <Text color="gray">
        {state.status === 'thinking' && '💭 Thinking...'}
        {state.status === 'tool' && `🔧 Using ${state.toolName}...`}
        {state.status === 'answering' && '✍️  Writing response...'}
      </Text>
    </Box>
  );
}
```

### 3. HistoryItemView Component

Displays past queries with expandable tool details:

```typescript
export function HistoryItemView({ item }: { item: HistoryItem }) {
  return (
    <Box flexDirection="column" marginBottom={1}>
      {/* User query */}
      <Text color="cyan">➤ {item.query}</Text>
      
      {/* Tool executions (if any) */}
      {item.events.map(event => (
        <AgentEventView key={event.id} event={event} />
      ))}
      
      {/* Answer */}
      {item.answer && (
        <Box marginTop={1}>
          <Text>{item.answer}</Text>
        </Box>
      )}
      
      {/* Status indicator */}
      {item.status === 'interrupted' && (
        <Text color="yellow">⚠️  Interrupted</Text>
      )}
    </Box>
  );
}
```

### 4. AgentEventView Component

Shows individual agent actions (thinking, tool calls):

```typescript
export function AgentEventView({ event }: { event: HistoryEvent }) {
  if (event.event.type === 'thinking') {
    return <Text color="gray" dimColor>💭 {event.event.message}</Text>;
  }
  
  if (event.event.type === 'tool_start') {
    return (
      <Box>
        <Text color="blue">
          🔧 {event.event.tool}({JSON.stringify(event.event.args)})
        </Text>
        {event.completed && event.endEvent && (
          <Text color="green"> ✓ {event.endEvent.duration}ms</Text>
        )}
      </Box>
    );
  }
  
  // ... other event types
}
```

## Key Code Sections Explained

### 1. Tool Execution with Error Handling

From `src/agent/agent.ts`:

```typescript
private async *executeToolCall(
  toolName: string,
  toolArgs: Record<string, unknown>,
  query: string,
  scratchpad: Scratchpad
): AsyncGenerator<ToolStartEvent | ToolEndEvent | ToolErrorEvent, void> {
  yield { type: 'tool_start', tool: toolName, args: toolArgs };

  const startTime = Date.now();

  try {
    // Invoke tool directly from toolMap
    const tool = this.toolMap.get(toolName);
    if (!tool) {
      throw new Error(`Tool '${toolName}' not found`);
    }
    const rawResult = await tool.invoke(
      toolArgs, 
      this.signal ? { signal: this.signal } : undefined
    );
    const result = typeof rawResult === 'string' ? rawResult : JSON.stringify(rawResult);
    const duration = Date.now() - startTime;

    yield { type: 'tool_end', tool: toolName, args: toolArgs, result, duration };

    // Generate LLM summary for context compaction
    const llmSummary = await this.summarizeToolResult(query, toolName, toolArgs, result);

    // Add complete tool result to scratchpad
    scratchpad.addToolResult(toolName, toolArgs, result, llmSummary);
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    yield { type: 'tool_error', tool: toolName, error: errorMessage };

    // Add error to scratchpad
    const toolDescription = getToolDescription(toolName, toolArgs);
    const errorSummary = `- ${toolDescription} [FAILED]: ${errorMessage}`;
    scratchpad.addToolResult(toolName, toolArgs, `Error: ${errorMessage}`, errorSummary);
  }
}
```

**Why this matters:**
- Errors don't crash the agent - they're treated as data
- Failed tool calls are recorded in scratchpad
- Agent can see the error and try a different approach
- UI shows failures gracefully

### 2. Streaming Answer Generation

From `src/agent/agent.ts`:

```typescript
private async *generateFinalAnswer(
  query: string,
  scratchpad: Scratchpad
): AsyncGenerator<AnswerStartEvent | AnswerChunkEvent> {
  yield { type: 'answer_start' };

  // Get full context data from scratchpad
  const fullContext = this.buildFullContextForAnswer(scratchpad);

  // Build the final answer prompt
  const prompt = buildFinalAnswerPrompt(query, fullContext);

  // Stream the final answer using provider-agnostic streaming
  const stream = streamLlmResponse(prompt, {
    model: this.model,
    systemPrompt: this.systemPrompt,
  });

  for await (const chunk of stream) {
    yield { type: 'answer_chunk', text: chunk };
  }
}
```

**Why this matters:**
- Users see answers as they're generated (feels fast)
- Works across all LLM providers (OpenAI, Anthropic, etc.)
- UI updates in real-time

### 3. Abort Signal Propagation

From `src/hooks/useAgentRunner.ts`:

```typescript
const runQuery = useCallback(async (query: string) => {
  // Create abort controller for this execution
  const abortController = new AbortController();
  abortControllerRef.current = abortController;
  
  try {
    const agent = await Agent.create({
      ...agentConfig,
      signal: abortController.signal,  // Pass to agent
    });
    const stream = agent.run(query, inMemoryChatHistoryRef.current!);
    
    for await (const event of stream) {
      handleEvent(event);
    }
  } catch (e) {
    // Handle abort gracefully
    if (e instanceof Error && e.name === 'AbortError') {
      setHistory(prev => {
        const last = prev[prev.length - 1];
        return [...prev.slice(0, -1), { ...last, status: 'interrupted' }];
      });
      return undefined;
    }
    // ... handle other errors
  }
}, [agentConfig]);

const cancelExecution = useCallback(() => {
  if (abortControllerRef.current) {
    abortControllerRef.current.abort();  // Signal all operations to stop
  }
}, []);
```

**Why this matters:**
- Users can cancel long-running queries with Esc or Ctrl+C
- Cancellation propagates through entire stack (agent → LLM → tools)
- Graceful shutdown without errors or undefined state

## System Prompt Architecture

The system prompt (`src/agent/prompts.ts`) is carefully crafted to guide the agent's behavior:

```typescript
export function buildSystemPrompt(model: string): string {
  const toolDescriptions = buildToolDescriptions(model);

  return `You are Dexter, a CLI assistant with access to research tools.

Current date: ${getCurrentDate()}

Your output is displayed on a command line interface. Keep responses short and concise.

## Available Tools

${toolDescriptions}

## Tool Usage Policy

- Only use tools when the query actually requires external data
- ALWAYS prefer financial_search over web_search for any financial data
- Call financial_search ONCE with the full natural language query
- Do NOT break up queries into multiple tool calls when one call can handle the request
- If a query can be answered from general knowledge, respond directly without using tools

## Behavior

- Prioritize accuracy over validation
- Use professional, objective tone
- For research tasks, be thorough but efficient

## Response Format

- Keep casual responses brief and direct
- For research: lead with the key finding and include specific data points
- For non-comparative information, prefer plain text or simple lists over tables
- Don't narrate your actions or ask leading questions

## Tables (for comparative/tabular data)

Tables render in a terminal with limited width. Keep them compact and scannable.

Structure:
- Max 4-6 columns per table
- Single-entity data: use vertical layout (metrics as rows)
- Multi-entity comparison: use horizontal layout (entities as columns)

Column headers and cell values must be short:
- Tickers not names: "AAPL" not "Apple Inc."
- Abbreviate metrics: Rev, Op Inc, Net Inc, OCF, FCF
- Numbers compact: 102.5B not $102,466,000,000
- Percentages: "31%" not "31.24%" unless precision matters
`;
}
```

**Key Insights:**
1. **Context awareness**: Current date is injected (helps with "recent" queries)
2. **Tool guidance**: Clear rules on when to use which tools
3. **Format instructions**: Optimized for terminal display
4. **Behavioral guidelines**: Sets tone and thoroughness expectations

## Performance Optimizations

### 1. Fast Models for Summarization

```typescript
const FAST_MODELS: Record<string, string> = {
  openai: 'gpt-4.1',
  anthropic: 'claude-haiku-4-5',
  google: 'gemini-3-flash-preview',
};

// Used for:
private async summarizeToolResult(...): Promise<string> {
  const summary = await callLlm(prompt, {
    model: getFastModel(this.modelProvider, this.model),  // Fast!
    systemPrompt: 'You are a concise data summarizer.',
  });
}
```

**Impact:** Summaries are 5-10x faster than using full reasoning models

### 2. Parallel Tool Execution

```typescript
// Execute multiple tools concurrently when possible
const results = await Promise.all(
  toolCalls.map(async (call) => {
    const tool = FINANCE_TOOL_MAP.get(call.name);
    return await tool.invoke(call.args);
  })
);
```

**Impact:** Multiple data sources fetched simultaneously

### 3. Context Window Management

Instead of:
```
Iteration 1 prompt: Query + Full Result 1 (10KB)
Iteration 2 prompt: Query + Full Result 1 (10KB) + Full Result 2 (10KB)
Iteration 3 prompt: Query + Full Result 1 (10KB) + Full Result 2 (10KB) + Full Result 3 (10KB)
Total: 60KB
```

Dexter does:
```
Iteration 1 prompt: Query + Summary 1 (100 bytes)
Iteration 2 prompt: Query + Summary 1 (100 bytes) + Summary 2 (100 bytes)
Iteration 3 prompt: Query + Summary 1 (100 bytes) + Summary 2 (100 bytes) + Summary 3 (100 bytes)
Final answer: Query + Full Result 1 (10KB) + Full Result 2 (10KB) + Full Result 3 (10KB)
Total: 30.6KB (50% reduction)
```

**Impact:** Can fit more iterations in context, faster processing, lower costs

### 4. Caching in Multi-Turn Conversations

```typescript
async selectRelevantMessages(currentQuery: string): Promise<Message[]> {
  // Check cache first - avoid redundant LLM calls
  const cacheKey = this.hashQuery(currentQuery);
  const cached = this.relevantMessagesByQuery.get(cacheKey);
  if (cached) return cached;
  
  // ... compute and cache
}
```

**Impact:** Same query in a session reuses relevance decisions

## Error Handling

Dexter has robust error handling at multiple levels:

### 1. Tool Execution Errors

Tools can fail (API errors, rate limits, network issues) - these are caught and treated as data:

```typescript
catch (error) {
  const errorMessage = error instanceof Error ? error.message : String(error);
  yield { type: 'tool_error', tool: toolName, error: errorMessage };

  // Error is added to scratchpad so agent can adapt
  scratchpad.addToolResult(toolName, toolArgs, `Error: ${errorMessage}`, errorSummary);
}
```

### 2. LLM Call Errors

LLM calls have retry logic with exponential backoff:

```typescript
async function withRetry<T>(fn: () => Promise<T>, maxAttempts = 3): Promise<T> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (e) {
      if (attempt === maxAttempts - 1) throw e;
      await new Promise((r) => setTimeout(r, 500 * 2 ** attempt));
    }
  }
  throw new Error('Unreachable');
}
```

Delays: 500ms → 1000ms → 2000ms

### 3. Agent Execution Errors

Top-level errors are caught and displayed to user:

```typescript
try {
  const agent = await Agent.create({ ...agentConfig, signal: abortController.signal });
  const stream = agent.run(query, inMemoryChatHistoryRef.current!);
  
  for await (const event of stream) {
    handleEvent(event);
  }
} catch (e) {
  if (e instanceof Error && e.name === 'AbortError') {
    // Graceful cancellation
    setHistory(prev => [...prev.slice(0, -1), { ...last, status: 'interrupted' }]);
  } else {
    // Real error
    const errorMsg = e instanceof Error ? e.message : String(e);
    setError(errorMsg);
    setHistory(prev => [...prev.slice(0, -1), { ...last, status: 'error' }]);
  }
}
```

## Configuration

Dexter is configured via environment variables (`.env` file):

```bash
# LLM Provider API Keys (choose one or more)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=AI...
XAI_API_KEY=xai-...

# For local models
OLLAMA_BASE_URL=http://127.0.0.1:11434

# Financial Data (required)
FINANCIAL_DATASETS_API_KEY=...

# Web Search (optional)
TAVILY_API_KEY=tvly-...
```

Model selection is done via the `/model` command in the CLI, which presents an interactive menu.

## File Structure Summary

```
dexter/
├── src/
│   ├── agent/              # Core agent logic
│   │   ├── agent.ts        # Main agent loop (312 lines)
│   │   ├── scratchpad.ts   # Memory/logging (177 lines)
│   │   ├── prompts.ts      # System & user prompts (193 lines)
│   │   └── types.ts        # Type definitions (100 lines)
│   │
│   ├── cli.tsx             # Main UI component (210 lines)
│   ├── index.tsx           # Entry point (12 lines)
│   │
│   ├── components/         # UI components (Ink/React)
│   │   ├── Input.tsx
│   │   ├── Intro.tsx
│   │   ├── WorkingIndicator.tsx
│   │   ├── HistoryItemView.tsx
│   │   ├── AgentEventView.tsx
│   │   └── ...
│   │
│   ├── hooks/              # React hooks
│   │   ├── useAgentRunner.ts      # Agent execution (236 lines)
│   │   ├── useModelSelection.ts
│   │   ├── useInputHistory.ts
│   │   └── ...
│   │
│   ├── model/              # LLM abstraction
│   │   └── llm.ts          # Multi-provider LLM interface
│   │
│   ├── tools/              # Tool implementations
│   │   ├── registry.ts     # Tool registration
│   │   ├── finance/        # Financial data tools
│   │   │   ├── financial-search.ts  # Router tool
│   │   │   ├── fundamentals.ts      # Income statements, etc.
│   │   │   ├── prices.ts            # Stock prices
│   │   │   ├── metrics.ts           # Financial ratios
│   │   │   └── ...
│   │   ├── search/         # Web search
│   │   └── descriptions/   # Tool descriptions for prompts
│   │
│   └── utils/              # Utilities
│       ├── in-memory-chat-history.ts  # Multi-turn context
│       ├── llm-stream.ts              # Streaming helpers
│       ├── ai-message.ts              # Message parsing
│       └── ...
│
├── package.json            # Dependencies
├── tsconfig.json           # TypeScript config
├── .env                    # API keys
└── README.md               # User documentation
```

## Conclusion

Dexter is a well-architected autonomous agent with several sophisticated features:

1. **ReAct Loop**: Thinks, acts, observes, repeats
2. **Context Compaction**: Manages limited context windows efficiently
3. **Streaming UX**: Real-time updates feel responsive
4. **Multi-Turn Conversations**: Maintains context across queries
5. **Robust Error Handling**: Gracefully handles failures
6. **Modular Architecture**: Easy to extend with new tools/providers
7. **Type Safety**: TypeScript throughout for reliability
8. **Professional UI**: Clean terminal interface with Ink

The codebase is clean, well-organized, and demonstrates advanced patterns in agent development, making it an excellent reference for building autonomous AI systems.
