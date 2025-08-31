# Dyad Agent Loops: How AI-Powered Code Generation Works

## Introduction

Dyad uses a sophisticated but intentionally simple agentic loop system to generate and modify code based on user requests. Unlike complex multi-agent systems that can become expensive and unpredictable, Dyad employs a streamlined approach that balances effectiveness with cost-efficiency. This document explains how the agent loops work, how models are selected, what tools are available, and how the entire system operates.

## Core Agent Loop Overview

Dyad's agent loop is designed to be **simple and predictable** rather than complex and potentially expensive. The system follows this basic flow:

1. **User Input** → User sends a prompt describing what they want to build or modify
2. **Model Selection** → Dyad selects an appropriate AI model based on user settings
3. **Context Preparation** → The entire codebase (or selected portions) is sent to the AI
4. **AI Response** → The model generates code changes using XML-like tags
5. **Response Processing** → Dyad parses and executes the AI's instructions
6. **Optional Auto-Fix Loop** → If enabled, TypeScript errors are automatically fixed

## Model Selection

### Available Model Types

Dyad supports multiple model providers and types:

- **Cloud Models**: OpenAI, Anthropic, Google, and other cloud providers
- **Local Models**: Ollama and LM Studio for local inference
- **Custom Models**: User-configured models with custom endpoints

### Model Selection Process

1. **User Configuration**: Users select their preferred model in the ModelPicker component
2. **Settings Storage**: The selected model is stored in user settings
3. **Runtime Selection**: During each request, Dyad uses the currently selected model
4. **Fallback Handling**: If a model becomes unavailable, Dyad gracefully handles errors

### Model Switching

Users can switch models at any time through the UI:
- **ModelPicker Component**: Dropdown interface for selecting models
- **Real-time Switching**: Changes take effect immediately for new requests
- **Provider Grouping**: Models are organized by provider for easy selection

## Available Tools (Dyad Tags)

Dyad uses XML-like tags instead of traditional function calling. This approach allows for multiple tool calls in a single response and avoids JSON formatting issues that can affect code quality.

### Core File Operations

#### `<dyad-write>` - Create or Update Files
```xml
<dyad-write path="src/components/Button.tsx" description="Creating a new button component">
import React from 'react';

const Button = ({ children, onClick }) => {
  return (
    <button onClick={onClick} className="px-4 py-2 bg-blue-500 text-white rounded">
      {children}
    </button>
  );
};

export default Button;
</dyad-write>
```

**How it works:**
- Parsed by `getDyadWriteTags()` function
- Extracts path, content, and optional description
- Creates new files or overwrites existing ones
- Handles code fence removal automatically

#### `<dyad-delete>` - Remove Files
```xml
<dyad-delete path="src/components/OldComponent.tsx"></dyad-delete>
```

**How it works:**
- Parsed by `getDyadDeleteTags()` function
- Removes specified files from the filesystem
- Updates any import references automatically

#### `<dyad-rename>` - Move/Rename Files
```xml
<dyad-rename from="src/components/UserProfile.tsx" to="src/components/ProfileCard.tsx"></dyad-rename>
```

**How it works:**
- Parsed by `getDyadRenameTags()` function
- Moves files from one location to another
- Updates import statements in other files

### Package Management

#### `<dyad-add-dependency>` - Install NPM Packages
```xml
<dyad-add-dependency packages="react-hot-toast lucide-react"></dyad-add-dependency>
```

**How it works:**
- Parsed by `getDyadAddDependencyTags()` function
- Splits package names by spaces
- Runs `npm install` for each package
- Handles both production and development dependencies

### Database Operations

#### `<dyad-execute-sql>` - Run SQL Queries
```xml
<dyad-execute-sql description="Creating users table">
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL
);
</dyad-execute-sql>
```

**How it works:**
- Parsed by `getDyadExecuteSqlTags()` function
- Executes SQL against connected databases
- Supports both Supabase and Neon databases
- Handles migrations and schema changes

### System Commands

#### `<dyad-command>` - Trigger System Actions
```xml
<dyad-command type="rebuild"></dyad-command>
<dyad-command type="restart"></dyad-command>
<dyad-command type="refresh"></dyad-command>
```

**How it works:**
- Parsed by `getDyadCommandTags()` function
- Triggers UI actions (rebuild, restart, refresh)
- Provides user feedback through action buttons

### Metadata Tags

#### `<dyad-chat-summary>` - Set Chat Title
```xml
<dyad-chat-summary>Adding authentication system</dyad-chat-summary>
```

**How it works:**
- Parsed by `getDyadChatSummaryTag()` function
- Sets the chat title for organization
- Used for chat history and navigation

## The Auto-Fix Loop

### How Auto-Fix Works

When "Auto-fix problems" is enabled, Dyad implements a sophisticated error correction loop:

1. **Initial Response**: AI generates code changes
2. **TypeScript Check**: Dyad runs TypeScript compiler to check for errors
3. **Error Detection**: If errors are found, they're formatted into a problem report
4. **Auto-Fix Attempt**: AI receives the error report and attempts to fix issues
5. **Iteration**: Process repeats up to 2 times maximum
6. **Success or Give Up**: Either errors are resolved or Dyad stops trying

### Auto-Fix Implementation

```typescript
// From chat_stream_handlers.ts
let autoFixAttempts = 0;
while (
  problemReport.problems.length > 0 &&
  autoFixAttempts < 2 &&
  !abortController.signal.aborted
) {
  // Generate problem fix prompt
  const problemFixPrompt = createProblemFixPrompt(problemReport);
  
  // Send to AI with context
  const { fullStream } = await simpleStreamText({
    modelClient,
    chatMessages: [
      // ... existing context
      { role: "user", content: problemFixPrompt }
    ]
  });
  
  autoFixAttempts++;
}
```

### Problem Report Format

Errors are formatted as XML for the AI:
```xml
<dyad-problem-report summary="3 problems">
<problem file="src/components/Button.tsx" line="5" column="12" code="TS2322">
Type 'string' is not assignable to type 'number'.
</problem>
<problem file="src/utils/helpers.ts" line="15" column="8" code="TS2304">
Cannot find name 'undefinedVariable'.
</problem>
</dyad-problem-report>
```

## Context Management

### Codebase Context

Dyad sends the entire codebase to the AI by default, but offers several context management strategies:

1. **Full Codebase**: Default approach for small projects
2. **Smart Context**: Uses smaller models to filter relevant files
3. **Manual Selection**: Users can manually select specific files
4. **Component Selection**: UI allows selecting specific components

### Context Engineering

The system prompt includes:
- **System Instructions**: How to use Dyad tags and respond
- **Tech Stack Information**: Available libraries and frameworks
- **File Structure**: Project organization and conventions
- **Coding Guidelines**: Best practices and patterns

## Response Processing Pipeline

### 1. Stream Processing
```typescript
// Real-time parsing of AI response
while (streaming) {
  const part = await stream.next();
  if (part.type === "text-delta") {
    fullResponse += part.text;
    // Update UI in real-time
    await processResponseChunkUpdate({ fullResponse });
  }
}
```

### 2. Tag Extraction
```typescript
// Parse all Dyad tags from response
const writeTags = getDyadWriteTags(fullResponse);
const renameTags = getDyadRenameTags(fullResponse);
const deletePaths = getDyadDeleteTags(fullResponse);
const addDependencies = getDyadAddDependencyTags(fullResponse);
```

### 3. Execution Order
1. **Dependencies**: Install packages first
2. **File Operations**: Delete → Rename → Write
3. **Database Operations**: Execute SQL queries
4. **System Commands**: Trigger UI actions

### 4. Virtual File System
Dyad uses a virtual file system to track changes before applying them:
```typescript
const virtualFileSystem = new AsyncVirtualFileSystem(appPath, {
  fileExists: (fileName) => fileExists(fileName),
  readFile: (fileName) => readFileWithCache(fileName),
});

virtualFileSystem.applyResponseChanges({
  deletePaths,
  renameTags,
  writeTags,
});
```

## Error Handling and Recovery

### Graceful Degradation
- **Model Failures**: Automatic fallback and error reporting
- **Network Issues**: Retry logic with exponential backoff
- **File System Errors**: Detailed error messages and recovery suggestions
- **Parse Errors**: Validation and error reporting for malformed tags

### User Feedback
- **Real-time Updates**: Streaming responses with live preview
- **Error Notifications**: Toast messages for important events
- **Progress Indicators**: Loading states and progress bars
- **Action Buttons**: Manual triggers for rebuild/restart/refresh

## Cost Optimization

### Why Simple Loops?

Dyad intentionally avoids complex agentic workflows for several reasons:

1. **Cost Control**: Complex loops can result in dozens of LLM calls per user request
2. **Predictability**: Simple loops are more reliable and easier to debug
3. **User Experience**: Faster responses and more predictable behavior
4. **Maintainability**: Simpler code is easier to maintain and improve

### Efficiency Features

- **Single Request**: Most operations complete in one AI call
- **Smart Context**: Only relevant files are sent when possible
- **Caching**: File system and TypeScript compilation caching
- **Streaming**: Real-time feedback without waiting for completion

## Future Considerations

### Potential Enhancements

1. **MCP Support**: Integration with Model Context Protocol for more sophisticated tool calling
2. **Multi-Agent Systems**: More complex workflows for advanced use cases
3. **Cost-Based Routing**: Automatic model selection based on task complexity
4. **Learning Loops**: AI that learns from user feedback and preferences

### Current Limitations

1. **Single Model**: Only one model used per request (no model switching mid-request)
2. **Limited Tool Set**: Focused on file operations and basic system commands
3. **No Planning**: No explicit planning phase before execution
4. **Fixed Retry Logic**: Auto-fix limited to 2 attempts

## Conclusion

Dyad's agent loop system prioritizes simplicity, reliability, and cost-effectiveness over complexity. By using XML-like tags instead of traditional function calling, implementing a focused auto-fix loop, and maintaining clear separation of concerns, Dyad provides a powerful yet predictable AI-powered development experience.

The system is designed to be transparent to users while providing sophisticated capabilities under the hood. This approach ensures that users can understand what's happening, predict costs, and maintain control over their development workflow.