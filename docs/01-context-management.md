# Context Management: How OpenCode Reads Code Without Exceeding LLM Context Limits

## Overview

OpenCode is an AI coding agent that needs to work with potentially large codebases while respecting the context window limits of LLMs (typically 128K-200K tokens). This document explains the strategies OpenCode uses to intelligently read and send only relevant code sections to the LLM API.

## The Challenge

Large codebases can have hundreds of thousands of lines of code. If we tried to send everything to an LLM:
- **Token limits**: Most LLMs have context windows of ~128K-200K tokens (roughly 100K-150K lines)
- **Cost**: More tokens = higher API costs
- **Performance**: Larger contexts can slow down LLM responses
- **Focus**: Too much irrelevant code can confuse the LLM

## OpenCode's Solution: Multi-Layered Strategy

OpenCode uses several complementary strategies to manage context efficiently:

---

## 1. Selective File Reading with Pagination

### How It Works

OpenCode provides a `read` tool that allows reading files with intelligent pagination:

```typescript
// From: packages/opencode/src/tool/read.ts
const DEFAULT_READ_LIMIT = 2000  // lines
const MAX_BYTES = 50 * 1024      // 50KB per read
```

**Key Features:**
- Reads files in chunks (default: 2000 lines)
- Has byte limits (50KB) to prevent overwhelming the context
- Supports `offset` and `limit` parameters for targeted reading
- Automatically truncates extremely long lines (>2000 chars)

**Example Usage:**
```typescript
// Read first 2000 lines
read({ filePath: "/path/to/file.ts" })

// Read lines 1000-1500
read({ filePath: "/path/to/file.ts", offset: 1000, limit: 500 })
```

### Why This Matters

Instead of reading entire files, OpenCode:
1. First reads a manageable chunk
2. Identifies relevant sections
3. Requests additional sections only if needed

---

## 2. Smart Search Before Reading

### Grep Tool - Pattern-Based Search

OpenCode uses `grep` (via ripgrep) to search for patterns BEFORE reading files:

```typescript
// From: packages/opencode/src/tool/grep.ts
grep({
  pattern: "function handleSubmit",
  path: "/src",
  include: "*.ts"
})
```

**Benefits:**
- Finds relevant files without reading them
- Returns only matching lines with context
- Filters by file patterns (e.g., only `.ts` files)

### Glob Tool - File Discovery

Finds files by name patterns:

```typescript
glob({
  pattern: "**/*.config.ts"
})
```

**Strategy:**
1. Use `glob` to find candidate files
2. Use `grep` to verify they contain relevant code
3. Use `read` to get full details only for confirmed matches

---

## 3. LSP Integration for Code Intelligence

OpenCode integrates with Language Server Protocol (LSP) to understand code structure without reading everything:

```typescript
// From: packages/opencode/src/lsp/
- Go to definition
- Find references
- Get diagnostics (errors/warnings)
- Symbol navigation
```

**How This Helps:**
- Jump directly to function definitions
- Find where variables/functions are used
- Understand code relationships without reading entire files

---

## 4. Truncation with Overflow Handling

When tool outputs are too large, OpenCode uses a truncation system:

```typescript
// From: packages/opencode/src/tool/truncation.ts
export const MAX_LINES = 2000
export const MAX_BYTES = 50 * 1024

// If output exceeds limits:
// 1. Show preview (first or last N lines/bytes)
// 2. Save full output to temporary file
// 3. Guide agent to use grep/read with offset to explore
```

**Example Truncation Message:**
```
[First 2000 lines shown]

...5000 lines truncated...

Full output saved to: /tmp/tool_abc123
Use Grep to search the full content or Read with offset/limit to view specific sections.
```

---

## 5. Context Compaction (Message History Pruning)

As conversations grow, the message history consumes context. OpenCode uses "compaction":

```typescript
// From: packages/opencode/src/session/compaction.ts

export const PRUNE_MINIMUM = 20_000    // tokens
export const PRUNE_PROTECT = 40_000    // tokens
```

**How It Works:**

1. **Overflow Detection:**
   ```typescript
   async function isOverflow(input: { tokens, model }) {
     const context = model.limit.context
     const count = tokens.input + tokens.cache.read + tokens.output
     const usable = context - output_buffer
     return count > usable
   }
   ```

2. **Pruning Strategy:**
   - Goes backwards through message history
   - Keeps recent turns (last 2 turns always protected)
   - Removes old tool call outputs (keeps input/metadata)
   - Protects important tools (e.g., "skill" tool)
   - Only prunes if saves >20K tokens

3. **Compaction Process:**
   - Creates a summary of conversation so far
   - LLM generates a detailed prompt describing:
     - What was accomplished
     - Current state
     - Next steps
     - Files being worked on
   - This summary replaces the long history

---

## 6. Token Estimation

OpenCode estimates token usage to make decisions:

```typescript
// From: packages/opencode/src/util/token.ts
const CHARS_PER_TOKEN = 4

export function estimate(input: string) {
  return Math.max(0, Math.round(input.length / 4))
}
```

This rough estimation (4 chars ≈ 1 token) helps decide when to:
- Truncate outputs
- Trigger compaction
- Read additional context

---

## 7. Caching with Prompt Caching

OpenCode leverages LLM provider caching features:

```typescript
// System prompts and early messages are cacheable
// This means:
// - System instructions are cached
// - Repeated file contents are cached
// - Reduces effective token usage
```

---

## Real-World Example: Editing a Function

Here's how OpenCode might modify a function in a large codebase:

### Step 1: Find the File
```typescript
grep({
  pattern: "function processPayment",
  path: "/src"
})
// Returns: src/payment/processor.ts:42: function processPayment()
```

### Step 2: Read Relevant Section
```typescript
read({
  filePath: "/src/payment/processor.ts",
  offset: 35,    // Read around line 42
  limit: 50      // Get context
})
```

### Step 3: Make the Edit
```typescript
edit({
  filePath: "/src/payment/processor.ts",
  oldString: "if (amount > 0) { ... }",
  newString: "if (amount > 0 && validated) { ... }"
})
```

**Total Context Used:**
- Grep output: ~200 tokens
- Read output: ~500 tokens (50 lines)
- Edit parameters: ~100 tokens
- **Total: ~800 tokens** vs. 10,000+ if entire file was read

---

## Advanced Techniques

### 1. Directory Listing (ls tool)
Lists directory structure without reading files:
```typescript
ls({ path: "/src", depth: 2 })
```

### 2. Code Search (External API)
For API/SDK documentation:
```typescript
codesearch({
  query: "React useState hook examples",
  tokensNum: 5000  // Controlled token limit
})
```

### 3. Batch Operations
Read multiple small files in parallel:
```typescript
// OpenCode can call tools in parallel
read({ filePath: "src/config.ts" })  // parallel
read({ filePath: "src/types.ts" })   // parallel
read({ filePath: "src/utils.ts" })   // parallel
```

---

## Configuration Options

Users can configure compaction behavior:

```typescript
// config file
{
  compaction: {
    auto: false,    // Disable automatic compaction
    prune: false    // Disable pruning
  }
}
```

---

## Key Takeaways

1. **Never read everything** - Use search tools first
2. **Read in chunks** - Use offset/limit for large files
3. **Truncate when necessary** - Show preview, save full output
4. **Prune old messages** - Remove outdated tool outputs
5. **Compact periodically** - Summarize long conversations
6. **Estimate tokens** - Make informed decisions about context usage
7. **Leverage caching** - Reuse repeated content efficiently

This multi-layered approach allows OpenCode to work with codebases of any size while staying within LLM context limits and keeping costs reasonable.

---

## See Also

- [Diff Strategy Documentation](./02-diff-strategy.md) - How OpenCode applies changes efficiently
- [Tool Reference](./03-tool-reference.md) - Complete tool documentation
