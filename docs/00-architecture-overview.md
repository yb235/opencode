# OpenCode Architecture Overview

## Introduction

This document provides a comprehensive overview of OpenCode's architecture, explaining how the different components work together to enable efficient AI-powered code editing while managing context limits and applying changes precisely.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        User Interface                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   TUI    │  │   Web    │  │ Desktop  │  │   API    │   │
│  │ (Terminal│  │   App    │  │   App    │  │  Server  │   │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└────────┼────────────┼─────────────┼─────────────┼──────────┘
         │            │             │             │
         └────────────┴─────────────┴─────────────┘
                      │
         ┌────────────▼────────────┐
         │   Session Manager       │
         │  - Message history      │
         │  - State management     │
         │  - Compaction           │
         └────────────┬────────────┘
                      │
         ┌────────────▼────────────┐
         │   LLM Integration       │
         │  - Provider abstraction │
         │  - Streaming responses  │
         │  - Token management     │
         └────────────┬────────────┘
                      │
         ┌────────────▼────────────┐
         │   Tool System           │
         │  - File operations      │
         │  - Code search          │
         │  - Diff/Patch           │
         │  - LSP integration      │
         └────────────┬────────────┘
                      │
         ┌────────────▼────────────┐
         │   File System           │
         │  - Version control      │
         │  - File tracking        │
         │  - Snapshots            │
         └─────────────────────────┘
```

---

## Core Components

### 1. Session Management

**Location**: `packages/opencode/src/session/`

**Purpose**: Manages conversation state and message history.

**Key Files:**
- `index.ts` - Session CRUD operations
- `processor.ts` - Processes LLM streaming responses
- `message-v2.ts` - Message type definitions
- `compaction.ts` - Context window management

**How It Works:**

```typescript
// Session structure
{
  id: "session_abc123",
  messages: [
    {
      role: "user",
      parts: [{ type: "text", text: "Add error handling" }]
    },
    {
      role: "assistant", 
      parts: [
        { type: "text", text: "I'll add try-catch..." },
        { type: "tool", tool: "edit", state: {...} }
      ],
      tokens: { input: 1500, output: 800 }
    }
  ]
}
```

**Key Responsibilities:**
1. Store and retrieve messages
2. Track token usage per message
3. Trigger compaction when context fills up
4. Manage conversation branches

---

### 2. LLM Integration

**Location**: `packages/opencode/src/session/llm.ts` and `packages/opencode/src/provider/`

**Purpose**: Abstract interface to different LLM providers.

**Supported Providers:**
- OpenAI (GPT-4, GPT-4 Turbo, etc.)
- Anthropic (Claude 3.5 Sonnet, Claude 3 Opus, etc.)
- Google (Gemini Pro, etc.)
- OpenRouter (any model)
- Local models via OpenAI-compatible APIs

**How It Works:**

```typescript
// LLM streaming interface
async function stream(input: {
  model: Provider.Model,
  messages: ModelMessage[],
  tools: Record<string, Tool>,
  system: string[],
  abort: AbortSignal
}) {
  // 1. Build system prompt
  const system = [
    SystemPrompt.header(),      // Core instructions
    agent.prompt,               // Agent-specific prompt
    ...customPrompts            // Additional context
  ]
  
  // 2. Apply provider-specific transformations
  const options = ProviderTransform.options(model)
  
  // 3. Stream response
  const stream = await streamText({
    model: language,
    messages: input.messages,
    system: system,
    tools: input.tools,
    ...options
  })
  
  // 4. Process streaming chunks
  for await (const chunk of stream.fullStream) {
    // Handle text, tool calls, reasoning, etc.
  }
}
```

**Provider Transformations:**
Different providers have different requirements:
- Token limits vary
- Tool calling formats differ
- Caching strategies differ
- Some support reasoning tokens

The transform layer normalizes these differences.

---

### 3. Tool System

**Location**: `packages/opencode/src/tool/`

**Purpose**: Provide LLM with abilities to interact with code and filesystem.

**Core Tools:**

#### File Operations
- **read**: Read files with pagination
- **write**: Create new files
- **edit**: Make surgical string replacements
- **multiedit**: Batch edits to same file
- **patch**: Multi-file changes

#### Code Search
- **grep**: Pattern-based code search
- **glob**: File name pattern matching
- **codesearch**: External API for SDK/library docs

#### Directory Operations
- **ls**: List directory contents

#### Language Intelligence
- **lsp**: Language server protocol integration
  - Go to definition
  - Find references
  - Diagnostics

#### Other
- **bash**: Execute shell commands
- **task**: Delegate to sub-agents
- **skill**: Execute predefined workflows
- **websearch/webfetch**: Internet access

**Tool Definition Pattern:**

```typescript
export const MyTool = Tool.define("mytool", {
  description: "What this tool does...",
  
  parameters: z.object({
    param1: z.string().describe("Description"),
    param2: z.number().optional()
  }),
  
  async execute(params, ctx) {
    // 1. Validate parameters
    // 2. Ask permission if needed
    await ctx.ask({
      permission: "mytool",
      patterns: [params.target]
    })
    
    // 3. Perform operation
    const result = await doWork(params)
    
    // 4. Return result
    return {
      title: "Success",
      output: "Operation completed",
      metadata: { ... }
    }
  }
})
```

**Tool Context (`ctx`):**
- `sessionID`: Current session
- `messageID`: Current message
- `ask()`: Request user permission
- `metadata()`: Attach metadata to response

---

### 4. Context Management System

**Location**: Multiple files working together

**Purpose**: Keep conversation within LLM context limits.

**Components:**

#### A. Token Estimation
```typescript
// packages/opencode/src/util/token.ts
const CHARS_PER_TOKEN = 4

function estimate(text: string): number {
  return Math.round(text.length / 4)
}
```

#### B. Truncation
```typescript
// packages/opencode/src/tool/truncation.ts
async function output(text: string, options: {
  maxLines?: number,
  maxBytes?: number,
  direction?: "head" | "tail"
}) {
  // If output too large:
  // 1. Show preview
  // 2. Save full output to temp file
  // 3. Guide LLM to use grep/read for exploration
}
```

#### C. Compaction
```typescript
// packages/opencode/src/session/compaction.ts

// 1. Detect overflow
if (await isOverflow({ tokens, model })) {
  // 2. Try pruning first
  await prune({ sessionID })
  
  // 3. If still too large, compact
  await process({
    sessionID,
    messages,
    auto: true
  })
}
```

**Compaction Process:**
1. Detect context overflow
2. Prune old tool outputs (saves 20K-40K tokens typically)
3. If still too large, create summary:
   - Send conversation to LLM
   - Ask: "Summarize our work for a new session"
   - LLM generates detailed context about:
     - What was accomplished
     - Current file state
     - Next steps planned
4. Replace message history with summary

---

### 5. Diff/Patch System

**Location**: `packages/opencode/src/tool/edit.ts` and `packages/opencode/src/patch/`

**Purpose**: Apply code changes efficiently without full file rewrites.

**Edit Tool Flow:**

```typescript
async function edit(params: {
  filePath: string,
  oldString: string,
  newString: string
}) {
  // 1. Lock file
  await FileTime.withLock(filePath, async () => {
    
    // 2. Read current content
    const contentOld = await readFile(filePath)
    
    // 3. Apply replacement with smart matching
    const contentNew = replace(
      contentOld,
      params.oldString,
      params.newString
    )
    
    // 4. Generate diff
    const diff = createTwoFilesPatch(
      filePath, filePath,
      contentOld, contentNew
    )
    
    // 5. Ask permission
    await ctx.ask({
      permission: "edit",
      metadata: { diff }
    })
    
    // 6. Write changes
    await writeFile(filePath, contentNew)
    
    // 7. Run LSP diagnostics
    await LSP.touchFile(filePath)
    const diagnostics = await LSP.diagnostics()
    
    // 8. Report errors if any
    if (diagnostics.errors.length > 0) {
      return { output: "File has errors, please fix..." }
    }
  })
}
```

**Smart Matching Strategies:**

The `replace()` function tries 9 different strategies:

1. **Exact match**: String appears as-is
2. **Line-trimmed**: Flexible end-of-line whitespace
3. **Block anchor**: Match on first/last lines, fuzzy middle
4. **Whitespace normalized**: Ignore spacing differences
5. **Indentation flexible**: Handle different indent levels
6. **Escape normalized**: Handle escaped characters
7. **Trimmed boundary**: Flexible surrounding whitespace
8. **Context aware**: Use surrounding code context
9. **Multi-occurrence**: Find all exact matches

This handles cases where LLM copies code with slight formatting differences.

---

### 6. File Tracking System

**Location**: `packages/opencode/src/file/time.ts` and `packages/opencode/src/snapshot/`

**Purpose**: Prevent edit conflicts and enable rollback.

**File Time Tracking:**

```typescript
// Track when file was read
FileTime.read(sessionID, filePath)

// Before editing, verify file hasn't changed
await FileTime.assert(sessionID, filePath)
// Throws error if file modified since last read

// After editing, update tracking
FileTime.read(sessionID, filePath)
```

**Snapshot System:**

Uses Git to track file state:

```typescript
// packages/opencode/src/snapshot/

// Create snapshot before changes
const hash = await track()

// Get changed files since snapshot  
const patch = await patch(hash)

// Restore to previous state
await restore(hash)

// Revert specific files
await revert(patches)
```

**Benefits:**
- Prevents conflicting edits
- Enables undo functionality
- Tracks what changed per session
- Supports rollback on errors

---

### 7. LSP Integration

**Location**: `packages/opencode/src/lsp/`

**Purpose**: Provide language intelligence without reading entire codebase.

**Capabilities:**

```typescript
// Go to definition
const definition = await LSP.definition(filePath, position)

// Find all references
const refs = await LSP.references(filePath, position)

// Get diagnostics (errors/warnings)
const diagnostics = await LSP.diagnostics()

// Get symbols in file
const symbols = await LSP.documentSymbol(filePath)

// Hover information
const hover = await LSP.hover(filePath, position)
```

**How It Helps:**
- Jump to function definitions without reading files
- Find where code is used
- Get compile/lint errors immediately after edits
- Navigate code structure intelligently

**Supported Languages:**
- TypeScript/JavaScript (ts-server)
- Python (pyright, pylsp)
- Go (gopls)
- Rust (rust-analyzer)
- And many more via standard LSP

---

### 8. Permission System

**Location**: `packages/opencode/src/permission/`

**Purpose**: Control what tools can do and ask user approval.

**How It Works:**

```typescript
// Agent permissions (defined in agent config)
{
  read: ["src/**"],        // Can read in src/
  edit: ["src/**"],        // Can edit in src/
  bash: "ask",             // Must ask before bash
  grep: "*"                // Can grep anywhere
}

// Tool requests permission
await ctx.ask({
  permission: "edit",
  patterns: ["src/api/handler.ts"],
  metadata: { diff: "..." }
})
```

**Permission Actions:**
- `"allow"`: Always permit
- `"deny"`: Always block  
- `"ask"`: Prompt user each time
- `"*"`: Allow all (wildcard)

**Use Cases:**
- Prevent agent from modifying config files
- Require approval for bash commands
- Allow unrestricted read but controlled write
- Block network access for security

---

## Data Flow: Complete Example

Let's trace a request through the system: **"Add error handling to the payment function"**

### Step 1: User Input
```
User (TUI) → Session Manager
Message: "Add error handling to the payment function"
```

### Step 2: Session Processing
```typescript
// Session Manager creates message
const userMessage = {
  role: "user",
  parts: [{ type: "text", text: "Add error..." }]
}

// Create assistant message placeholder
const assistantMessage = {
  role: "assistant",
  parts: [],
  tokens: { input: 0, output: 0 }
}
```

### Step 3: LLM Streaming
```typescript
// Session Manager → LLM Integration
const stream = await LLM.stream({
  model: claudeSonnet,
  messages: conversationHistory,
  tools: { read, edit, grep, ... },
  system: [agentPrompt, toolDescriptions]
})
```

### Step 4: LLM Decides to Search
```typescript
// LLM Response (streamed):
{
  type: "tool-call",
  tool: "grep",
  args: {
    pattern: "function.*payment",
    include: "*.ts"
  }
}
```

### Step 5: Tool Execution
```typescript
// Processor → Tool System
const result = await GrepTool.execute({
  pattern: "function.*payment",
  include: "*.ts"
}, ctx)

// Returns:
// src/payment/processor.ts:42: function processPayment()
```

### Step 6: LLM Reads File
```typescript
// LLM Response:
{
  type: "tool-call",
  tool: "read",
  args: {
    filePath: "src/payment/processor.ts",
    offset: 35,
    limit: 50
  }
}
```

### Step 7: Tool Returns Content
```typescript
// ReadTool.execute() returns:
{
  output: `
35. // Process payment transaction
36. async function processPayment(req) {
37.   const amount = req.body.amount;
38.   const result = await chargeCustomer(amount);
39.   return { success: true, result };
40. }
  `
}
```

### Step 8: LLM Generates Edit
```typescript
// LLM Response:
{
  type: "tool-call",
  tool: "edit",
  args: {
    filePath: "src/payment/processor.ts",
    oldString: `async function processPayment(req) {
  const amount = req.body.amount;
  const result = await chargeCustomer(amount);
  return { success: true, result };
}`,
    newString: `async function processPayment(req) {
  try {
    const amount = req.body.amount;
    const result = await chargeCustomer(amount);
    return { success: true, result };
  } catch (error) {
    console.error("Payment failed:", error);
    return { success: false, error: error.message };
  }
}`
  }
}
```

### Step 9: Edit Tool Execution
```typescript
// EditTool.execute():

// 1. Lock file
// 2. Read current content
// 3. Find oldString using smart matching
// 4. Replace with newString
// 5. Generate diff
const diff = `
@@ -36,5 +36,11 @@
 async function processPayment(req) {
+  try {
   const amount = req.body.amount;
   const result = await chargeCustomer(amount);
   return { success: true, result };
+  } catch (error) {
+    console.error("Payment failed:", error);
+    return { success: false, error: error.message };
+  }
 }
`

// 6. Ask permission (show diff to user)
await ctx.ask({ permission: "edit", metadata: { diff } })

// 7. Write file
await writeFile(filePath, newContent)

// 8. Check LSP diagnostics
const diagnostics = await LSP.diagnostics()
```

### Step 10: Response to User
```
Assistant (TUI) ← Session Manager

✓ Searched for payment function
✓ Read src/payment/processor.ts
✓ Added try-catch error handling

Diff:
[Shows unified diff]

No syntax errors detected.
```

### Step 11: Token Accounting
```typescript
// Session Manager updates message tokens
assistantMessage.tokens = {
  input: 2500,   // System prompt + history + tool results
  output: 800,   // LLM generated text + tool calls
  cache: {
    read: 1500,  // Cached system prompt reused
    write: 0
  }
}

// Check if approaching context limit
if (await isOverflow({ tokens, model })) {
  // Trigger compaction for next turn
}
```

---

## Token Usage Breakdown

For the above example:

### Input Tokens (~2500)
- System prompt (cached): 1500 tokens
- Agent instructions: 300 tokens
- Conversation history: 500 tokens
- Tool results (grep + read): 200 tokens

### Output Tokens (~800)
- Reasoning text: 400 tokens
- Tool call parameters: 400 tokens

### Total Cost
With Claude Sonnet 3.5:
- Input: 2500 × $3/M = $0.0075
- Output: 800 × $15/M = $0.012
- Cached: 1500 × $0.30/M = $0.00045
- **Total: ~$0.02 per turn**

---

## Performance Optimizations

### 1. Prompt Caching
System prompts are marked as cacheable:
```typescript
const system = [
  cacheableHeader,  // Never changes
  agentPrompt       // Rarely changes
]
```
First request: Pay full price
Subsequent: 90% discount on cached content

### 2. Parallel Tool Calls
LLM can call multiple tools at once:
```typescript
// Single response with multiple tools
await Promise.all([
  read({ filePath: "src/types.ts" }),
  read({ filePath: "src/utils.ts" }),
  read({ filePath: "src/config.ts" })
])
```

### 3. Streaming Responses
User sees progress immediately:
- Text appears as generated
- Tool calls execute as received
- No waiting for full response

### 4. Incremental Diagnostics
LSP runs in background:
- Doesn't block tool execution
- Results shown when available
- Cached for subsequent checks

---

## Error Handling & Recovery

### Edit Failures
```typescript
try {
  await edit({ oldString: "...", newString: "..." })
} catch (error) {
  if (error.message.includes("not found")) {
    // Re-read file to get current content
    await read({ filePath })
  } else if (error.message.includes("multiple matches")) {
    // Provide more context in oldString
  }
}
```

### Context Overflow
```typescript
// Automatic handling
if (await SessionCompaction.isOverflow({ tokens, model })) {
  await SessionCompaction.prune({ sessionID })
  if (await SessionCompaction.isOverflow({ tokens, model })) {
    await SessionCompaction.process({ sessionID, auto: true })
  }
}
```

### File Conflicts
```typescript
// Edit fails if file changed since read
try {
  await FileTime.assert(sessionID, filePath)
} catch (error) {
  // File was modified - re-read required
  return "File was modified since last read. Please read it again."
}
```

### Network Errors
```typescript
// LLM API call failed
try {
  const stream = await LLM.stream({ ... })
} catch (error) {
  // Retry with exponential backoff
  await SessionRetry.process({ sessionID })
}
```

---

## Configuration System

**Location**: `packages/opencode/src/config/config.ts`

Users can configure behavior:

```typescript
// ~/.opencode/config.json
{
  "compaction": {
    "auto": true,      // Auto-compact when full
    "prune": true      // Prune old tool outputs
  },
  "snapshot": true,    // Enable git snapshots
  "lsp": {
    "enabled": true,
    "languages": {
      "typescript": "tsserver",
      "python": "pyright"
    }
  }
}
```

---

## Agent System

**Location**: `packages/opencode/src/agent/`

OpenCode supports multiple agent personalities:

### Built-in Agents

1. **build** (default)
   - Full file system access
   - Can edit files freely
   - Executes bash commands
   - Best for development work

2. **plan** (read-only)
   - Read-only file access
   - Asks before bash commands
   - Best for code exploration
   - Won't make accidental changes

3. **compaction**
   - Special agent for summarizing
   - Creates context summaries
   - Used internally

**Agent Configuration:**
```typescript
// .opencode/agents/myagent.json
{
  "name": "myagent",
  "model": {
    "providerID": "anthropic",
    "modelID": "claude-3-5-sonnet"
  },
  "prompt": "Custom agent instructions...",
  "permission": {
    "read": ["src/**"],
    "edit": ["src/**", "!src/critical/**"],
    "bash": "ask"
  },
  "options": {
    "temperature": 0.7
  }
}
```

---

## Plugin System

**Location**: `packages/opencode/src/plugin/`

Extend OpenCode with custom functionality:

```typescript
export const MyPlugin = Plugin.create({
  name: "my-plugin",
  
  async onSessionStart(ctx) {
    // Run when session starts
  },
  
  async onToolExecute(ctx, tool, params) {
    // Intercept tool calls
  },
  
  async onCompacting(ctx) {
    // Modify compaction process
    return {
      context: ["additional context..."],
      prompt: "custom compaction prompt"
    }
  }
})
```

---

## Key Design Principles

### 1. Token Efficiency
Every design decision considers token usage:
- Read files in chunks, not entirely
- Use diffs instead of full rewrites
- Truncate large outputs
- Compact when necessary

### 2. Precision Over Speed
Better to make correct changes slowly than fast mistakes:
- Smart string matching with fallbacks
- LSP validation after edits
- File locking prevents conflicts
- Permission system gates dangerous operations

### 3. Composable Tools
Small, focused tools that combine well:
- grep → read → edit (find, examine, modify)
- glob → read (discover, examine)
- ls → grep → read (explore, search, examine)

### 4. Progressive Enhancement
Works without advanced features:
- Core functionality without LSP
- Manual compaction if auto disabled
- Works with any LLM provider
- Graceful degradation

### 5. Observable & Debuggable
Everything is tracked and visible:
- Token usage per message
- Full message history
- Diffs show all changes
- Snapshots enable rollback
- Logs for troubleshooting

---

## Summary

OpenCode achieves efficient AI coding through:

1. **Smart Context Management**
   - Selective reading (grep → read)
   - Pagination for large files
   - Truncation with overflow files
   - Compaction when needed

2. **Efficient Change Application**
   - Diff strategy over full rewrites
   - Multiple matching strategies
   - Multi-file atomic changes
   - LSP validation

3. **Safety & Reliability**
   - File locking
   - Permission system
   - Snapshot/rollback capability
   - Error recovery

4. **Extensibility**
   - Multiple LLM providers
   - Custom agents
   - Plugin system
   - Configurable behavior

This architecture allows OpenCode to work with codebases of any size while staying within LLM context limits and applying changes precisely and safely.

---

## Further Reading

- [Context Management Documentation](./01-context-management.md)
- [Diff Strategy Documentation](./02-diff-strategy.md)
- [Tool Reference](./03-tool-reference.md) (if created)
- [Agent Configuration Guide](./04-agent-guide.md) (if created)
