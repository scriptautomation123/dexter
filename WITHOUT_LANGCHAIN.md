# How to Write Dexter Without LangChain

This document explains how Dexter could be rewritten without using LangChain, providing direct API implementations for each major LLM provider.

## Table of Contents

1. [Current LangChain Usage](#current-langchain-usage)
2. [Why Remove LangChain?](#why-remove-langchain)
3. [Implementation Alternatives](#implementation-alternatives)
4. [Step-by-Step Migration Guide](#step-by-step-migration-guide)
5. [Code Examples](#code-examples)
6. [Trade-offs](#trade-offs)

## Current LangChain Usage

Dexter currently uses LangChain for:

1. **Multi-provider LLM Abstractions** - Unified interface for OpenAI, Anthropic, Google, xAI, and Ollama
2. **Tool/Function Calling** - Structured tool definitions and execution
3. **Structured Output** - Schema validation with Zod
4. **Streaming Responses** - Provider-agnostic streaming
5. **Prompt Templates** - Message formatting

### Dependencies

```json
{
  "@langchain/anthropic": "^1.1.3",
  "@langchain/core": "^1.1.0",
  "@langchain/google-genai": "^2.0.0",
  "@langchain/ollama": "^1.0.3",
  "@langchain/openai": "^1.1.3",
  "@langchain/tavily": "^1.0.1"
}
```

## Why Remove LangChain?

Potential reasons to consider removing LangChain:

- **Reduce bundle size** - LangChain adds significant dependencies
- **Direct control** - Full control over API calls and error handling
- **Performance** - Eliminate abstraction overhead
- **Simplicity** - Fewer dependencies to maintain
- **Customization** - Easier to implement provider-specific features
- **Debugging** - Simpler stack traces

## Implementation Alternatives

### 1. Direct API Integration

Each LLM provider offers official SDKs or REST APIs:

| Provider | SDK/API |
|----------|---------|
| OpenAI | `openai` (official SDK) |
| Anthropic | `@anthropic-ai/sdk` (official SDK) |
| Google | `@google/generative-ai` (official SDK) |
| xAI | OpenAI-compatible REST API |
| Ollama | REST API |

### 2. Unified Interface Pattern

Create a common interface for all providers:

```typescript
interface ChatModel {
  invoke(messages: Message[], options?: CallOptions): Promise<AIMessage>;
  stream(messages: Message[], options?: CallOptions): AsyncIterator<string>;
}

interface CallOptions {
  tools?: Tool[];
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
}
```

## Step-by-Step Migration Guide

### Step 1: Install Provider SDKs

```bash
# Remove LangChain
bun remove @langchain/openai @langchain/anthropic @langchain/google-genai @langchain/ollama @langchain/core @langchain/tavily

# Add provider SDKs
bun add openai @anthropic-ai/sdk @google/generative-ai
```

### Step 2: Create Base Interfaces

```typescript
// src/model/types.ts

export interface Message {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

export interface Tool {
  name: string;
  description: string;
  parameters: {
    type: 'object';
    properties: Record<string, unknown>;
    required?: string[];
  };
}

export interface ToolCall {
  id: string;
  name: string;
  arguments: Record<string, unknown>;
}

export interface AIMessage {
  content: string;
  toolCalls?: ToolCall[];
}

export interface ModelConfig {
  model: string;
  temperature?: number;
  maxTokens?: number;
  streaming?: boolean;
}
```

### Step 3: Implement OpenAI Provider

```typescript
// src/model/providers/openai.ts

import OpenAI from 'openai';
import type { Message, Tool, AIMessage, ModelConfig } from '../types.js';

export class OpenAIProvider {
  private client: OpenAI;
  
  constructor(apiKey: string, baseURL?: string) {
    this.client = new OpenAI({ apiKey, baseURL });
  }
  
  async invoke(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): Promise<AIMessage> {
    const response = await this.client.chat.completions.create({
      model: config.model,
      messages: messages.map(m => ({
        role: m.role,
        content: m.content,
      })),
      temperature: config.temperature,
      max_tokens: config.maxTokens,
      tools: tools ? tools.map(t => ({
        type: 'function' as const,
        function: {
          name: t.name,
          description: t.description,
          parameters: t.parameters,
        },
      })) : undefined,
    }, { signal });
    
    const message = response.choices[0].message;
    
    return {
      content: message.content || '',
      toolCalls: message.tool_calls?.map(tc => ({
        id: tc.id,
        name: tc.function.name,
        arguments: JSON.parse(tc.function.arguments),
      })),
    };
  }
  
  async *stream(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): AsyncIterableIterator<string> {
    const stream = await this.client.chat.completions.create({
      model: config.model,
      messages: messages.map(m => ({
        role: m.role,
        content: m.content,
      })),
      temperature: config.temperature,
      max_tokens: config.maxTokens,
      tools: tools ? tools.map(t => ({
        type: 'function' as const,
        function: {
          name: t.name,
          description: t.description,
          parameters: t.parameters,
        },
      })) : undefined,
      stream: true,
    }, { signal });
    
    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content;
      if (content) {
        yield content;
      }
    }
  }
}
```

### Step 4: Implement Anthropic Provider

```typescript
// src/model/providers/anthropic.ts

import Anthropic from '@anthropic-ai/sdk';
import type { Message, Tool, AIMessage, ModelConfig } from '../types.js';

export class AnthropicProvider {
  private client: Anthropic;
  
  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey });
  }
  
  async invoke(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): Promise<AIMessage> {
    // Anthropic requires separate system message
    const systemMessage = messages.find(m => m.role === 'system')?.content;
    const userMessages = messages.filter(m => m.role !== 'system');
    
    const response = await this.client.messages.create({
      model: config.model,
      system: systemMessage,
      messages: userMessages.map(m => ({
        role: m.role as 'user' | 'assistant',
        content: m.content,
      })),
      max_tokens: config.maxTokens || 4096,
      temperature: config.temperature,
      tools: tools ? tools.map(t => ({
        name: t.name,
        description: t.description,
        input_schema: t.parameters,
      })) : undefined,
    }, { signal });
    
    let content = '';
    let toolCalls: ToolCall[] | undefined;
    
    for (const block of response.content) {
      if (block.type === 'text') {
        content += block.text;
      } else if (block.type === 'tool_use') {
        if (!toolCalls) toolCalls = [];
        toolCalls.push({
          id: block.id,
          name: block.name,
          arguments: block.input,
        });
      }
    }
    
    return { content, toolCalls };
  }
  
  async *stream(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): AsyncIterableIterator<string> {
    const systemMessage = messages.find(m => m.role === 'system')?.content;
    const userMessages = messages.filter(m => m.role !== 'system');
    
    const stream = await this.client.messages.stream({
      model: config.model,
      system: systemMessage,
      messages: userMessages.map(m => ({
        role: m.role as 'user' | 'assistant',
        content: m.content,
      })),
      max_tokens: config.maxTokens || 4096,
      temperature: config.temperature,
      tools: tools ? tools.map(t => ({
        name: t.name,
        description: t.description,
        input_schema: t.parameters,
      })) : undefined,
    }, { signal });
    
    for await (const event of stream) {
      if (event.type === 'content_block_delta' && 
          event.delta.type === 'text_delta') {
        yield event.delta.text;
      }
    }
  }
}
```

### Step 5: Implement Google Provider

```typescript
// src/model/providers/google.ts

import { GoogleGenerativeAI } from '@google/generative-ai';
import type { Message, Tool, AIMessage, ModelConfig } from '../types.js';

export class GoogleProvider {
  private client: GoogleGenerativeAI;
  
  constructor(apiKey: string) {
    this.client = new GoogleGenerativeAI(apiKey);
  }
  
  async invoke(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): Promise<AIMessage> {
    const model = this.client.getGenerativeModel({ 
      model: config.model,
      systemInstruction: messages.find(m => m.role === 'system')?.content,
    });
    
    const userMessages = messages
      .filter(m => m.role !== 'system')
      .map(m => ({
        role: m.role === 'assistant' ? 'model' : 'user',
        parts: [{ text: m.content }],
      }));
    
    const result = await model.generateContent({
      contents: userMessages,
      generationConfig: {
        temperature: config.temperature,
        maxOutputTokens: config.maxTokens,
      },
      tools: tools ? [{
        functionDeclarations: tools.map(t => ({
          name: t.name,
          description: t.description,
          parameters: t.parameters,
        })),
      }] : undefined,
    });
    
    const response = result.response;
    const functionCalls = response.functionCalls();
    
    return {
      content: response.text() || '',
      toolCalls: functionCalls?.map((fc, i) => ({
        id: `call_${i}`,
        name: fc.name,
        arguments: fc.args,
      })),
    };
  }
  
  async *stream(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): AsyncIterableIterator<string> {
    const model = this.client.getGenerativeModel({ 
      model: config.model,
      systemInstruction: messages.find(m => m.role === 'system')?.content,
    });
    
    const userMessages = messages
      .filter(m => m.role !== 'system')
      .map(m => ({
        role: m.role === 'assistant' ? 'model' : 'user',
        parts: [{ text: m.content }],
      }));
    
    const result = await model.generateContentStream({
      contents: userMessages,
      generationConfig: {
        temperature: config.temperature,
        maxOutputTokens: config.maxTokens,
      },
    });
    
    for await (const chunk of result.stream) {
      const text = chunk.text();
      if (text) {
        yield text;
      }
    }
  }
}
```

### Step 6: Implement Ollama Provider

```typescript
// src/model/providers/ollama.ts

import type { Message, Tool, AIMessage, ModelConfig } from '../types.js';

export class OllamaProvider {
  private baseUrl: string;
  
  constructor(baseUrl = 'http://127.0.0.1:11434') {
    this.baseUrl = baseUrl;
  }
  
  async invoke(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): Promise<AIMessage> {
    const response = await fetch(`${this.baseUrl}/api/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: config.model,
        messages,
        options: {
          temperature: config.temperature,
          num_predict: config.maxTokens,
        },
        tools: tools?.map(t => ({
          type: 'function',
          function: {
            name: t.name,
            description: t.description,
            parameters: t.parameters,
          },
        })),
        stream: false,
      }),
      signal,
    });
    
    if (!response.ok) {
      throw new Error(`Ollama API error: ${response.statusText}`);
    }
    
    const data = await response.json();
    const message = data.message;
    
    return {
      content: message.content || '',
      toolCalls: message.tool_calls?.map((tc: any, i: number) => ({
        id: `call_${i}`,
        name: tc.function.name,
        arguments: tc.function.arguments,
      })),
    };
  }
  
  async *stream(
    messages: Message[],
    config: ModelConfig,
    tools?: Tool[],
    signal?: AbortSignal
  ): AsyncIterableIterator<string> {
    const response = await fetch(`${this.baseUrl}/api/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: config.model,
        messages,
        options: {
          temperature: config.temperature,
          num_predict: config.maxTokens,
        },
        stream: true,
      }),
      signal,
    });
    
    if (!response.ok) {
      throw new Error(`Ollama API error: ${response.statusText}`);
    }
    
    const reader = response.body?.getReader();
    if (!reader) {
      throw new Error('No response body');
    }
    
    const decoder = new TextDecoder();
    
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const text = decoder.decode(value);
        const lines = text.split('\n').filter(line => line.trim());
        
        for (const line of lines) {
          try {
            const data = JSON.parse(line);
            if (data.message?.content) {
              yield data.message.content;
            }
          } catch {
            // Skip invalid JSON lines
          }
        }
      }
    } finally {
      reader.releaseLock();
    }
  }
}
```

### Step 7: Create Unified Model Factory

```typescript
// src/model/llm.ts (replacing LangChain version)

import { OpenAIProvider } from './providers/openai.js';
import { AnthropicProvider } from './providers/anthropic.js';
import { GoogleProvider } from './providers/google.js';
import { OllamaProvider } from './providers/ollama.js';
import type { Message, Tool, AIMessage, ModelConfig } from './types.js';

type Provider = OpenAIProvider | AnthropicProvider | GoogleProvider | OllamaProvider;

export class ChatModel {
  private provider: Provider;
  private config: ModelConfig;
  
  constructor(modelName: string, streaming = false) {
    this.config = { model: modelName, streaming };
    this.provider = this.createProvider(modelName);
  }
  
  private createProvider(modelName: string): Provider {
    if (modelName.startsWith('claude-')) {
      const apiKey = process.env.ANTHROPIC_API_KEY;
      if (!apiKey) throw new Error('ANTHROPIC_API_KEY not found');
      return new AnthropicProvider(apiKey);
    }
    
    if (modelName.startsWith('gemini-')) {
      const apiKey = process.env.GOOGLE_API_KEY;
      if (!apiKey) throw new Error('GOOGLE_API_KEY not found');
      return new GoogleProvider(apiKey);
    }
    
    if (modelName.startsWith('grok-')) {
      const apiKey = process.env.XAI_API_KEY;
      if (!apiKey) throw new Error('XAI_API_KEY not found');
      return new OpenAIProvider(apiKey, 'https://api.x.ai/v1');
    }
    
    if (modelName.startsWith('ollama:')) {
      const baseUrl = process.env.OLLAMA_BASE_URL || 'http://127.0.0.1:11434';
      const actualModel = modelName.replace(/^ollama:/, '');
      this.config.model = actualModel;
      return new OllamaProvider(baseUrl);
    }
    
    // Default to OpenAI
    const apiKey = process.env.OPENAI_API_KEY;
    if (!apiKey) throw new Error('OPENAI_API_KEY not found');
    return new OpenAIProvider(apiKey);
  }
  
  async invoke(
    messages: Message[],
    tools?: Tool[],
    signal?: AbortSignal
  ): Promise<AIMessage> {
    return this.provider.invoke(messages, this.config, tools, signal);
  }
  
  async *stream(
    messages: Message[],
    tools?: Tool[],
    signal?: AbortSignal
  ): AsyncIterableIterator<string> {
    yield* this.provider.stream(messages, this.config, tools, signal);
  }
}

// Convenience function similar to LangChain's callLlm
export async function callLlm(
  systemPrompt: string,
  userPrompt: string,
  options: {
    model?: string;
    tools?: Tool[];
    signal?: AbortSignal;
  } = {}
): Promise<AIMessage> {
  const model = new ChatModel(options.model || 'gpt-5.2');
  
  const messages: Message[] = [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: userPrompt },
  ];
  
  return model.invoke(messages, options.tools, options.signal);
}

export async function* streamLlm(
  systemPrompt: string,
  userPrompt: string,
  options: {
    model?: string;
    signal?: AbortSignal;
  } = {}
): AsyncIterableIterator<string> {
  const model = new ChatModel(options.model || 'gpt-5.2', true);
  
  const messages: Message[] = [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: userPrompt },
  ];
  
  yield* model.stream(messages, undefined, options.signal);
}
```

### Step 8: Update Tool Definitions

```typescript
// src/tools/types.ts (without LangChain)

export interface ToolParameter {
  type: string;
  description?: string;
  enum?: string[];
  default?: unknown;
}

export interface ToolDefinition {
  name: string;
  description: string;
  parameters: {
    type: 'object';
    properties: Record<string, ToolParameter>;
    required?: string[];
  };
  func: (args: Record<string, unknown>) => Promise<string>;
}

// Helper to create tools
export function createTool(definition: ToolDefinition): ToolDefinition {
  return definition;
}
```

### Step 9: Update Tool Implementations

```typescript
// src/tools/finance/prices.ts (without LangChain)

import { createTool } from '../types.js';

export const getPriceSnapshot = createTool({
  name: 'get_price_snapshot',
  description: 'Fetches the most recent price snapshot for a specific stock ticker.',
  parameters: {
    type: 'object',
    properties: {
      ticker: {
        type: 'string',
        description: "The stock ticker symbol (e.g., 'AAPL' for Apple)",
      },
    },
    required: ['ticker'],
  },
  func: async (args) => {
    const { ticker } = args;
    const params = { ticker: String(ticker) };
    const { data, url } = await callApi('/prices/snapshot/', params);
    return formatToolResult(data.snapshot || {}, [url]);
  },
});

export const getPrices = createTool({
  name: 'get_prices',
  description: 'Retrieves historical price data for a stock over a date range.',
  parameters: {
    type: 'object',
    properties: {
      ticker: {
        type: 'string',
        description: "Stock ticker symbol (e.g., 'AAPL')",
      },
      interval: {
        type: 'string',
        enum: ['minute', 'day', 'week', 'month', 'year'],
        default: 'day',
        description: "Time interval for price data",
      },
      interval_multiplier: {
        type: 'number',
        default: 1,
        description: 'Multiplier for the interval',
      },
      start_date: {
        type: 'string',
        description: 'Start date in YYYY-MM-DD format',
      },
      end_date: {
        type: 'string',
        description: 'End date in YYYY-MM-DD format',
      },
    },
    required: ['ticker', 'start_date', 'end_date'],
  },
  func: async (args) => {
    const params = {
      ticker: String(args.ticker),
      interval: String(args.interval || 'day'),
      interval_multiplier: Number(args.interval_multiplier || 1),
      start_date: String(args.start_date),
      end_date: String(args.end_date),
    };
    const { data, url } = await callApi('/prices/', params);
    return formatToolResult(data.prices || [], [url]);
  },
});
```

### Step 10: Update Agent to Use New Interfaces

```typescript
// src/agent/agent.ts (key changes)

import { ChatModel, callLlm, streamLlm } from '../model/llm.js';
import type { Message, Tool, AIMessage } from '../model/types.js';

export class Agent {
  private model: ChatModel;
  private tools: Map<string, ToolDefinition>;
  
  constructor(config: AgentConfig) {
    this.model = new ChatModel(config.model || 'gpt-5.2');
    this.tools = new Map(getTools().map(t => [t.name, t]));
  }
  
  private async callModel(messages: Message[]): Promise<AIMessage> {
    const tools = Array.from(this.tools.values()).map(t => ({
      name: t.name,
      description: t.description,
      parameters: t.parameters,
    }));
    
    return this.model.invoke(messages, tools, this.signal);
  }
  
  private async executeToolCall(
    toolName: string,
    toolArgs: Record<string, unknown>
  ): Promise<string> {
    const tool = this.tools.get(toolName);
    if (!tool) {
      throw new Error(`Tool '${toolName}' not found`);
    }
    
    return tool.func(toolArgs);
  }
  
  private async *streamFinalAnswer(
    messages: Message[]
  ): AsyncIterableIterator<string> {
    yield* this.model.stream(messages, undefined, this.signal);
  }
}
```

## Code Examples

### Example: Making a Simple Call

**With LangChain:**

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate } from '@langchain/core/prompts';

const model = new ChatOpenAI({ model: 'gpt-4' });
const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'You are a helpful assistant.'],
  ['user', '{input}'],
]);
const chain = prompt.pipe(model);
const response = await chain.invoke({ input: 'Hello!' });
```

**Without LangChain:**

```typescript
import { OpenAI } from 'openai';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const response = await client.chat.completions.create({
  model: 'gpt-4',
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Hello!' },
  ],
});
const content = response.choices[0].message.content;
```

### Example: Streaming

**With LangChain:**

```typescript
const stream = await chain.stream({ input: 'Hello!' });
for await (const chunk of stream) {
  if (chunk.content) {
    process.stdout.write(chunk.content);
  }
}
```

**Without LangChain:**

```typescript
const stream = await client.chat.completions.create({
  model: 'gpt-4',
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Hello!' },
  ],
  stream: true,
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) {
    process.stdout.write(content);
  }
}
```

### Example: Tool/Function Calling

**With LangChain:**

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { z } from 'zod';

const tool = new DynamicStructuredTool({
  name: 'get_weather',
  description: 'Get the weather for a location',
  schema: z.object({
    location: z.string().describe('City name'),
  }),
  func: async ({ location }) => {
    return `The weather in ${location} is sunny`;
  },
});

const modelWithTools = model.bindTools([tool]);
const response = await modelWithTools.invoke('What is the weather in Paris?');
```

**Without LangChain:**

```typescript
const tools = [{
  type: 'function' as const,
  function: {
    name: 'get_weather',
    description: 'Get the weather for a location',
    parameters: {
      type: 'object',
      properties: {
        location: {
          type: 'string',
          description: 'City name',
        },
      },
      required: ['location'],
    },
  },
}];

const response = await client.chat.completions.create({
  model: 'gpt-4',
  messages: [
    { role: 'user', content: 'What is the weather in Paris?' },
  ],
  tools,
});

const toolCall = response.choices[0].message.tool_calls?.[0];
if (toolCall) {
  const args = JSON.parse(toolCall.function.arguments);
  const result = `The weather in ${args.location} is sunny`;
}
```

### Example: Structured Output

**With LangChain:**

```typescript
import { z } from 'zod';

const schema = z.object({
  revenue: z.number(),
  year: z.number(),
});

const modelWithOutput = model.withStructuredOutput(schema);
const result = await modelWithOutput.invoke('What was Apple revenue in 2023?');
```

**Without LangChain:**

```typescript
import { zodToJsonSchema } from 'zod-to-json-schema';
import { z } from 'zod';

const schema = z.object({
  revenue: z.number(),
  year: z.number(),
});

const response = await client.chat.completions.create({
  model: 'gpt-4',
  messages: [
    { role: 'user', content: 'What was Apple revenue in 2023?' },
  ],
  response_format: { 
    type: 'json_schema',
    json_schema: {
      name: 'revenue_data',
      schema: zodToJsonSchema(schema),
    },
  },
});

const result = JSON.parse(response.choices[0].message.content || '{}');
```

## Trade-offs

### Advantages of Removing LangChain

✅ **Smaller Bundle Size**
- Remove ~10+ dependencies
- Reduce bundle size by 30-50%

✅ **Better Performance**
- No abstraction overhead
- Direct API calls

✅ **Full Control**
- Custom error handling
- Provider-specific optimizations
- Easier debugging

✅ **Simpler Dependencies**
- Fewer breaking changes
- Easier to update providers independently

### Disadvantages of Removing LangChain

❌ **More Code to Maintain**
- Need to implement providers yourself
- Handle API changes manually

❌ **Loss of Features**
- LangChain provides many utilities (memory, callbacks, etc.)
- Would need to reimplement

❌ **Provider Compatibility**
- Must handle differences between providers
- More testing required

❌ **Less Standardization**
- Each provider has different interfaces
- More code to keep providers aligned

## Recommended Approach

For Dexter specifically:

1. **Keep LangChain if:**
   - You plan to add more LangChain features (agents, chains, memory)
   - You want to support many more providers easily
   - Bundle size is not a concern
   - You value standardization over control

2. **Remove LangChain if:**
   - Bundle size is critical
   - You only need 3-4 providers
   - You want maximum control over API calls
   - You're comfortable maintaining provider code

## Migration Checklist

If you decide to remove LangChain, follow this checklist:

- [ ] Install provider SDKs (openai, @anthropic-ai/sdk, @google/generative-ai)
- [ ] Create base type definitions (Message, Tool, AIMessage, etc.)
- [ ] Implement OpenAI provider
- [ ] Implement Anthropic provider  
- [ ] Implement Google provider
- [ ] Implement Ollama provider
- [ ] Create unified ChatModel class
- [ ] Update tool definitions to new format
- [ ] Update agent.ts to use new interfaces
- [ ] Update llm.ts to use new implementations
- [ ] Update llm-stream.ts for direct streaming
- [ ] Test each provider thoroughly
- [ ] Update tests
- [ ] Remove LangChain dependencies
- [ ] Update documentation

## Conclusion

Removing LangChain is **feasible** but requires significant refactoring. The official provider SDKs provide all the functionality Dexter needs, but you'll need to implement a unified interface layer yourself.

**Recommendation:** For most use cases, keeping LangChain provides good value through standardization and reduced maintenance burden. Consider removing it only if bundle size or direct API control are critical requirements.

The choice depends on your specific priorities:
- **Prioritize simplicity & bundle size** → Remove LangChain
- **Prioritize features & maintainability** → Keep LangChain
