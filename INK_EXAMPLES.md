# React-Based CLI with Ink: Simple to Complex Examples

This guide demonstrates how [Ink](https://github.com/vadimdemedes/ink) is used in Dexter to build a powerful React-based command-line interface. We'll progress from simple concepts to complex patterns used in production.

## Table of Contents
1. [What is Ink?](#what-is-ink)
2. [Level 1: Basic Ink Components](#level-1-basic-ink-components)
3. [Level 2: Interactive Input and Keyboard Handling](#level-2-interactive-input-and-keyboard-handling)
4. [Level 3: State Management with Hooks](#level-3-state-management-with-hooks)
5. [Level 4: Multi-Component Orchestration](#level-4-multi-component-orchestration)
6. [Level 5: Advanced Patterns - Real-Time Updates](#level-5-advanced-patterns---real-time-updates)

---

## What is Ink?

**Ink** is a React renderer for the command line. It lets you build and test CLI apps the same way you would build web apps with React.

**Key Benefits:**
- ✅ Use familiar React concepts (components, props, hooks, state)
- ✅ Build complex interactive CLIs with composable components
- ✅ Handle keyboard input elegantly
- ✅ Real-time UI updates and streaming
- ✅ Test your CLI with React testing patterns

---

## Level 1: Basic Ink Components

### 1.1 Hello World - The Simplest Ink App

```typescript
// src/index.tsx (simplified)
import React from 'react';
import { render, Text } from 'ink';

// Simple component
function HelloWorld() {
  return <Text>Hello from Ink!</Text>;
}

// Render to terminal
render(<HelloWorld />);
```

**Key Concepts:**
- `render()` - Renders React component to terminal (like ReactDOM.render)
- `<Text>` - Basic building block for displaying text

### 1.2 Styling with Box and Text

Dexter's **Intro component** shows how to style text with colors and layout:

```typescript
// src/components/Intro.tsx (simplified)
import React from 'react';
import { Box, Text } from 'ink';

export function Intro({ provider, model }) {
  return (
    <Box flexDirection="column" marginTop={2}>
      {/* Colored border */}
      <Text color="cyan">{'═'.repeat(60)}</Text>
      
      {/* Bold text with color */}
      <Text color="cyan">
        ║ <Text bold>Welcome to Dexter</Text> v3.0.2 ║
      </Text>
      
      <Text color="cyan">{'═'.repeat(60)}</Text>
      
      {/* ASCII art banner */}
      <Box marginTop={1}>
        <Text color="cyan" bold>
{`██████╗ ███████╗██╗  ██╗████████╗███████╗██████╗ 
██╔══██╗██╔════╝╚██╗██╔╝╚══██╔══╝██╔════╝██╔══██╗
██║  ██║█████╗   ╚███╔╝    ██║   █████╗  ██████╔╝`}
        </Text>
      </Box>
      
      {/* Dynamic content */}
      <Box marginY={1} flexDirection="column">
        <Text>Your AI assistant for financial research.</Text>
        <Text color="gray">
          Current model: <Text color="cyan">{model}</Text>
        </Text>
      </Box>
    </Box>
  );
}
```

**Key Concepts:**
- `<Box>` - Container for layout (flexbox-based)
- `flexDirection` - Control layout direction (column/row)
- `marginTop`, `marginY` - Spacing control
- `color`, `bold` - Text styling
- Dynamic content via props (`{model}`)

### 1.3 Animated Components with Spinner

```typescript
// src/components/WorkingIndicator.tsx (simplified)
import React from 'react';
import { Box, Text } from 'ink';
import Spinner from 'ink-spinner';

export function WorkingIndicator({ state }) {
  return (
    <Box>
      <Text color="cyan">
        <Spinner type="dots" />
      </Text>
      <Text color="gray"> Thinking... (esc to interrupt)</Text>
    </Box>
  );
}
```

**Key Concepts:**
- `ink-spinner` - Pre-built animated spinner component
- Combine components for rich UI

---

## Level 2: Interactive Input and Keyboard Handling

### 2.1 Basic Text Input

```typescript
// Using ink-text-input (third-party)
import React, { useState } from 'react';
import { Box, Text } from 'ink';
import TextInput from 'ink-text-input';

function SimpleInput() {
  const [query, setQuery] = useState('');
  
  return (
    <Box>
      <Text>Enter query: </Text>
      <TextInput 
        value={query}
        onChange={setQuery}
        onSubmit={() => console.log('Submitted:', query)}
      />
    </Box>
  );
}
```

### 2.2 Custom Input with Cursor - The Dexter Way

Dexter implements a **custom input component** with advanced features:

```typescript
// src/components/Input.tsx (simplified)
import React from 'react';
import { Box, Text, useInput } from 'ink';
import { useTextBuffer } from '../hooks/useTextBuffer.js';

export function Input({ onSubmit, historyValue, onHistoryNavigate }) {
  // Custom hook manages text buffer and cursor position
  const { text, cursorPosition, actions } = useTextBuffer();

  // useInput - Ink's hook for capturing keyboard input
  useInput((input, key) => {
    // Handle history navigation (up/down arrows)
    if (key.upArrow) {
      onHistoryNavigate('up');
      return;
    }
    if (key.downArrow) {
      onHistoryNavigate('down');
      return;
    }

    // Cursor movement - left arrow
    if (key.leftArrow && !key.ctrl && !key.meta) {
      actions.moveCursor(cursorPosition - 1);
      return;
    }

    // Cursor movement - right arrow
    if (key.rightArrow && !key.ctrl && !key.meta) {
      actions.moveCursor(cursorPosition + 1);
      return;
    }

    // Emacs-style shortcuts
    if (key.ctrl && input === 'a') {
      actions.moveCursor(0); // Move to start
      return;
    }
    if (key.ctrl && input === 'e') {
      actions.moveCursor(text.length); // Move to end
      return;
    }

    // Word navigation (Option+Arrow on Mac, Ctrl+Arrow on Windows)
    if ((key.meta && key.leftArrow) || (key.ctrl && key.leftArrow)) {
      actions.moveCursor(/* word backward logic */);
      return;
    }

    // Delete
    if (key.backspace || key.delete) {
      actions.deleteBackward();
      return;
    }

    // Submit on Enter
    if (key.return) {
      const val = text.trim();
      if (val) {
        onSubmit(val);
        actions.clear();
      }
      return;
    }

    // Regular character input
    if (input && !key.ctrl && !key.meta) {
      actions.insert(input);
    }
  });

  return (
    <Box borderStyle="single" borderColor="gray">
      <Box paddingX={1}>
        <Text color="cyan" bold>{'> '}</Text>
        <CursorText text={text} cursorPosition={cursorPosition} />
      </Box>
    </Box>
  );
}
```

**Key Concepts:**
- `useInput(callback)` - Captures all keyboard input
- `input` - Character typed (e.g., 'a', 'b', '1')
- `key` object - Special keys (arrows, ctrl, meta, return, etc.)
- Custom text buffer management for cursor control
- Emacs-style keyboard shortcuts (Ctrl+A, Ctrl+E)
- Word-level navigation (Option/Ctrl + Arrows)

### 2.3 Custom Cursor Display

```typescript
// src/components/CursorText.tsx
import React from 'react';
import { Text } from 'ink';

export function CursorText({ text, cursorPosition }) {
  const beforeCursor = text.substring(0, cursorPosition);
  const atCursor = text[cursorPosition] || ' ';
  const afterCursor = text.substring(cursorPosition + 1);

  return (
    <Text>
      {beforeCursor}
      <Text inverse>{atCursor}</Text>
      {afterCursor}
    </Text>
  );
}
```

**Key Concepts:**
- `inverse` - Inverts colors (creates cursor effect)
- String slicing to position cursor
- Render cursor at any position in text

---

## Level 3: State Management with Hooks

### 3.1 Custom Hook for Text Buffer

Dexter uses custom hooks to manage complex state:

```typescript
// src/hooks/useTextBuffer.ts (simplified)
import { useState, useCallback } from 'react';

export function useTextBuffer() {
  const [text, setText] = useState('');
  const [cursorPosition, setCursorPosition] = useState(0);

  const actions = {
    insert: useCallback((char: string) => {
      setText(prev => 
        prev.slice(0, cursorPosition) + char + prev.slice(cursorPosition)
      );
      setCursorPosition(prev => prev + 1);
    }, [cursorPosition]),

    deleteBackward: useCallback(() => {
      if (cursorPosition > 0) {
        setText(prev => 
          prev.slice(0, cursorPosition - 1) + prev.slice(cursorPosition)
        );
        setCursorPosition(prev => prev - 1);
      }
    }, [cursorPosition]),

    moveCursor: useCallback((newPos: number) => {
      setCursorPosition(Math.max(0, Math.min(newPos, text.length)));
    }, [text.length]),

    clear: useCallback(() => {
      setText('');
      setCursorPosition(0);
    }, []),

    setValue: useCallback((newText: string) => {
      setText(newText);
      setCursorPosition(newText.length);
    }, []),
  };

  return { text, cursorPosition, actions };
}
```

**Key Concepts:**
- Custom hooks encapsulate complex logic
- `useCallback` optimizes performance
- Immutable state updates
- Return actions as API

### 3.2 State Management for Agent Execution

```typescript
// src/hooks/useAgentRunner.ts (simplified)
import { useState, useRef, useCallback } from 'react';

export function useAgentRunner({ model, modelProvider, maxIterations }) {
  // State for UI
  const [history, setHistory] = useState([]);
  const [workingState, setWorkingState] = useState({ status: 'idle' });
  const [error, setError] = useState(null);
  
  // Ref for cancellation
  const abortControllerRef = useRef(null);
  
  const runQuery = useCallback(async (query: string) => {
    // Create abort controller for cancellation
    const abortController = new AbortController();
    abortControllerRef.current = abortController;
    
    // Add query to history immediately
    const itemId = Date.now().toString();
    setHistory(prev => [...prev, {
      id: itemId,
      query,
      events: [],
      answer: '',
      status: 'processing',
      startTime: Date.now(),
    }]);
    
    setWorkingState({ status: 'thinking' });
    
    try {
      // Create and run agent
      const agent = await Agent.create({
        model,
        modelProvider,
        maxIterations,
        signal: abortController.signal,
      });
      
      const stream = agent.run(query);
      
      // Process streaming results
      for await (const event of stream) {
        if (event.type === 'tool_start') {
          setWorkingState({ status: 'tool', toolName: event.toolName });
        } else if (event.type === 'tool_end') {
          // Add tool result to history
          setHistory(prev => prev.map(item =>
            item.id === itemId
              ? { ...item, events: [...item.events, event] }
              : item
          ));
        } else if (event.type === 'answer') {
          setWorkingState({ status: 'answering' });
          // Update answer in real-time
          setHistory(prev => prev.map(item =>
            item.id === itemId
              ? { ...item, answer: event.content }
              : item
          ));
        }
      }
      
      setWorkingState({ status: 'idle' });
      return { answer: /* final answer */ };
      
    } catch (err) {
      setError(err.message);
      setWorkingState({ status: 'idle' });
    }
  }, [model, modelProvider, maxIterations]);
  
  const cancelExecution = useCallback(() => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
      setWorkingState({ status: 'idle' });
    }
  }, []);
  
  return {
    history,
    workingState,
    error,
    isProcessing: workingState.status !== 'idle',
    runQuery,
    cancelExecution,
  };
}
```

**Key Concepts:**
- Multiple state variables for different concerns
- `useRef` for mutable values (abort controller)
- `useCallback` for stable function references
- Async operations with state updates
- Real-time streaming updates
- Cancellation support

---

## Level 4: Multi-Component Orchestration

### 4.1 Main CLI Component - Bringing It All Together

```typescript
// src/cli.tsx (simplified structure)
import React, { useCallback } from 'react';
import { Box, Text, useApp, useInput } from 'ink';

export function CLI() {
  const { exit } = useApp();
  
  // Model selection state (custom hook)
  const {
    selectionState,
    provider,
    model,
    startSelection,
    cancelSelection,
    handleProviderSelect,
    handleModelSelect,
    isInSelectionFlow,
  } = useModelSelection();
  
  // Agent execution state (custom hook)
  const {
    history,
    workingState,
    error,
    isProcessing,
    runQuery,
    cancelExecution,
  } = useAgentRunner({ model, modelProvider: provider, maxIterations: 10 });
  
  // Input history for navigation (custom hook)
  const {
    historyValue,
    navigateUp,
    navigateDown,
    saveMessage,
    resetNavigation,
  } = useInputHistory();
  
  // Handle user input submission
  const handleSubmit = useCallback(async (query: string) => {
    // Handle special commands
    if (query === 'exit' || query === 'quit') {
      console.log('Goodbye!');
      exit();
      return;
    }
    
    if (query === '/model') {
      startSelection();
      return;
    }
    
    // Ignore if busy
    if (isInSelectionFlow() || workingState.status !== 'idle') return;
    
    // Save to history and run query
    await saveMessage(query);
    resetNavigation();
    
    const result = await runQuery(query);
  }, [exit, startSelection, isInSelectionFlow, workingState, runQuery]);
  
  // Global keyboard shortcuts
  useInput((input, key) => {
    // Escape - cancel current operation
    if (key.escape) {
      if (isInSelectionFlow()) {
        cancelSelection();
      } else if (isProcessing) {
        cancelExecution();
      }
    }
    
    // Ctrl+C - cancel or exit
    if (key.ctrl && input === 'c') {
      if (isInSelectionFlow() || isProcessing) {
        cancelSelection();
        cancelExecution();
      } else {
        console.log('\nGoodbye!');
        exit();
      }
    }
  });
  
  // Render different screens based on state
  if (selectionState.appState === 'provider_select') {
    return <ProviderSelector onSelect={handleProviderSelect} />;
  }
  
  if (selectionState.appState === 'model_select') {
    return <ModelSelector onSelect={handleModelSelect} />;
  }
  
  // Main chat interface
  return (
    <Box flexDirection="column">
      <Intro provider={provider} model={model} />
      
      {/* Render conversation history */}
      {history.map(item => (
        <HistoryItemView key={item.id} item={item} />
      ))}
      
      {/* Error display */}
      {error && (
        <Box marginBottom={1}>
          <Text color="red">Error: {error}</Text>
        </Box>
      )}
      
      {/* Working indicator */}
      {isProcessing && <WorkingIndicator state={workingState} />}
      
      {/* Input box */}
      <Box marginTop={1}>
        <Input 
          onSubmit={handleSubmit}
          historyValue={historyValue}
          onHistoryNavigate={(dir) => 
            dir === 'up' ? navigateUp() : navigateDown()
          }
        />
      </Box>
    </Box>
  );
}
```

**Key Concepts:**
- **Composition**: Multiple custom hooks for separation of concerns
- **State machines**: Different UI states (provider selection, model selection, chat)
- **Global keyboard handling**: `useInput` at top level for shortcuts
- **Conditional rendering**: Different screens based on state
- **Event callbacks**: Props drilling for communication
- **Real-time updates**: UI updates as agent works

### 4.2 History Item View - Rendering Complex Data

```typescript
// src/components/HistoryItemView.tsx (simplified)
import React from 'react';
import { Box, Text } from 'ink';

export function HistoryItemView({ item }) {
  return (
    <Box flexDirection="column" marginY={1}>
      {/* User query */}
      <Box marginBottom={1}>
        <Text bold color="cyan">You: </Text>
        <Text>{item.query}</Text>
      </Box>
      
      {/* Tool calls/events */}
      {item.events.map((event, idx) => (
        <Box key={idx} marginLeft={2} marginBottom={1}>
          {event.type === 'tool_start' && (
            <Text color="yellow">
              🔧 Using tool: {event.toolName}
            </Text>
          )}
          {event.type === 'tool_end' && (
            <Box flexDirection="column">
              <Text color="green">✓ Tool result:</Text>
              <Text color="gray">{event.result}</Text>
            </Box>
          )}
        </Box>
      ))}
      
      {/* Agent answer */}
      {item.answer && (
        <Box marginTop={1}>
          <Text bold color="green">Dexter: </Text>
          <Text>{item.answer}</Text>
        </Box>
      )}
      
      {/* Status indicator */}
      {item.status === 'processing' && (
        <Text color="gray">Processing...</Text>
      )}
    </Box>
  );
}
```

**Key Concepts:**
- Nested `<Box>` components for structure
- Mapping over arrays to render lists
- Conditional rendering with `&&`
- Rich formatting with colors and emojis

---

## Level 5: Advanced Patterns - Real-Time Updates

### 5.1 Streaming Updates from Async Generators

One of Dexter's most powerful features is real-time streaming updates:

```typescript
// src/hooks/useAgentRunner.ts (streaming section)
export function useAgentRunner(agentConfig) {
  const runQuery = useCallback(async (query: string) => {
    // ... setup code ...
    
    const agent = await Agent.create(agentConfig);
    const stream = agent.run(query); // Returns AsyncGenerator
    
    // Process streaming events in real-time
    for await (const event of stream) {
      // Tool execution started
      if (event.type === 'tool_start') {
        setWorkingState({ 
          status: 'tool', 
          toolName: event.toolName 
        });
      }
      
      // Tool completed - add result to history
      else if (event.type === 'tool_end') {
        setHistory(prev => prev.map(item =>
          item.id === currentItemId
            ? { 
                ...item, 
                events: [...item.events, {
                  type: 'tool_result',
                  toolName: event.toolName,
                  result: event.result,
                  timestamp: Date.now(),
                }]
              }
            : item
        ));
      }
      
      // Answer streaming - update incrementally
      else if (event.type === 'answer_chunk') {
        setWorkingState({ status: 'answering' });
        setHistory(prev => prev.map(item =>
          item.id === currentItemId
            ? { 
                ...item, 
                answer: item.answer + event.content // Append chunk
              }
            : item
        ));
      }
      
      // Final answer
      else if (event.type === 'answer_complete') {
        setHistory(prev => prev.map(item =>
          item.id === currentItemId
            ? { 
                ...item, 
                status: 'complete',
                endTime: Date.now(),
              }
            : item
        ));
      }
    }
    
    setWorkingState({ status: 'idle' });
  }, [agentConfig]);
  
  return { runQuery, history, workingState };
}
```

**Key Concepts:**
- **Async generators**: `for await...of` loop for streaming
- **Incremental updates**: Append chunks instead of replacing
- **State machine updates**: Change working state based on events
- **Immutable updates**: Use `.map()` to update specific items
- **Real-time UI**: Ink re-renders on each state change

### 5.2 Cancellation Support with AbortController

```typescript
export function useAgentRunner(agentConfig) {
  const abortControllerRef = useRef<AbortController | null>(null);
  
  const runQuery = useCallback(async (query: string) => {
    // Create abort controller
    const abortController = new AbortController();
    abortControllerRef.current = abortController;
    
    try {
      const agent = await Agent.create({
        ...agentConfig,
        signal: abortController.signal, // Pass abort signal
      });
      
      const stream = agent.run(query);
      
      for await (const event of stream) {
        // Check if aborted
        if (abortController.signal.aborted) {
          break;
        }
        // ... process events ...
      }
    } catch (err) {
      if (err.name === 'AbortError') {
        setError('Operation cancelled by user');
      }
    }
  }, [agentConfig]);
  
  const cancelExecution = useCallback(() => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
      setWorkingState({ status: 'idle' });
      setError('Cancelled');
    }
  }, []);
  
  return { runQuery, cancelExecution };
}
```

**Key Concepts:**
- `useRef` holds mutable abort controller
- `AbortController` for graceful cancellation
- Pass `signal` to async operations
- Catch `AbortError` for user-friendly messages
- Clean up state on cancellation

### 5.3 Application Lifecycle with useApp

```typescript
import { useApp } from 'ink';

export function CLI() {
  const { exit } = useApp();
  
  // Exit on command
  const handleSubmit = useCallback((query: string) => {
    if (query === 'exit' || query === 'quit') {
      console.log('Goodbye!');
      exit(); // Cleanly exit Ink app
      return;
    }
  }, [exit]);
  
  // Ctrl+C handling
  useInput((input, key) => {
    if (key.ctrl && input === 'c') {
      console.log('\nGoodbye!');
      exit();
    }
  });
  
  return <Box>...</Box>;
}
```

**Key Concepts:**
- `useApp()` provides app lifecycle methods
- `exit()` cleanly shuts down Ink
- Call `console.log()` before exit for final output

---

## Summary: Key Patterns Used in Dexter

### Architecture Patterns
1. **Custom Hooks**: Encapsulate complex state logic (`useAgentRunner`, `useModelSelection`, `useTextBuffer`)
2. **Composition**: Build complex UI from small, focused components
3. **State Machines**: Manage different app states (selection flows, chat, processing)
4. **Event-Driven**: Use callbacks and events for component communication

### Ink-Specific Patterns
1. **Layout with Box**: Use flexbox for responsive layouts
2. **Styled Text**: Colors, bold, inverse for visual hierarchy
3. **useInput Hook**: Global and local keyboard handling
4. **useApp Hook**: App lifecycle management
5. **Conditional Rendering**: Show/hide components based on state

### Advanced Patterns
1. **Real-Time Streaming**: Update UI incrementally with async generators
2. **Cancellation**: AbortController for user cancellation
3. **Input History**: Navigate previous commands with arrow keys
4. **Custom Cursor**: Implement rich text editing features
5. **Multi-Screen Flows**: State-based navigation between screens

---

## Best Practices

### Do's ✅
- Use `useCallback` for event handlers to prevent re-renders
- Use `useRef` for mutable values that don't trigger renders
- Keep components small and focused
- Use custom hooks to share logic
- Provide keyboard shortcuts (Ctrl+C, Escape, etc.)
- Handle errors gracefully

### Don'ts ❌
- Don't use `console.log()` during rendering (corrupts display)
- Don't mutate state directly (use setState)
- Don't forget to clean up subscriptions/timers
- Don't block the event loop with synchronous operations
- Don't forget to handle edge cases (empty input, cancellation)

---

## Resources

- **Ink Documentation**: https://github.com/vadimdemedes/ink
- **Dexter Architecture**: [ARCHITECTURE.md](./ARCHITECTURE.md)
- **Code Walkthrough**: [CODE_WALKTHROUGH.md](./CODE_WALKTHROUGH.md)
- **Ink Component Gallery**: https://github.com/vadimdemedes/ink#useful-components

---

## Next Steps

To build your own React-based CLI:

1. **Start Simple**: Create a basic component with Text and Box
2. **Add Interaction**: Use `useInput` to handle keyboard
3. **Add State**: Use `useState` and custom hooks for logic
4. **Add Effects**: Use `useEffect` for side effects
5. **Compose**: Build complex UIs from small components
6. **Polish**: Add colors, spacing, animations

Happy building! 🚀
