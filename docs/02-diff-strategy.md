# Diff Strategy: How OpenCode Applies Code Changes Efficiently

## Overview

When an AI needs to modify code, there are two main approaches:
1. **Full File Replacement**: Send the entire new file content
2. **Diff/Patch Strategy**: Send only the changes (what to add, remove, or replace)

OpenCode uses a sophisticated diff/patch strategy that dramatically reduces token usage and makes changes more precise and reliable.

## The Problem with Full File Replacement

Imagine a 1000-line file where you need to change one function:

**Full Replacement Approach:**
```
Token usage: ~1000 lines × 4 tokens/line = ~4000 tokens (output)
Risk: LLM might introduce unintended changes to other parts
Risk: If LLM truncates output, file gets corrupted
```

**Diff Strategy:**
```
Token usage: ~20 lines × 4 tokens/line = ~80 tokens (output)
Precision: Only specified lines are changed
Safety: If something goes wrong, only the targeted section is affected
```

---

## OpenCode's Multi-Tool Diff Strategy

OpenCode provides three complementary tools for making changes:

---

## 1. Edit Tool - Surgical String Replacement

### How It Works

The `edit` tool performs exact string replacements:

```typescript
// From: packages/opencode/src/tool/edit.ts

edit({
  filePath: "/path/to/file.ts",
  oldString: "const x = 5;",
  newString: "const x = 10;",
  replaceAll: false  // optional: replace all occurrences
})
```

### Smart Matching System

OpenCode uses **9 different matching strategies** to find `oldString`:

1. **SimpleReplacer**: Exact match
2. **LineTrimmedReplacer**: Matches with flexible line-end whitespace
3. **BlockAnchorReplacer**: Matches based on first/last line anchors
4. **WhitespaceNormalizedReplacer**: Ignores extra spaces/tabs
5. **IndentationFlexibleReplacer**: Handles different indentation levels
6. **EscapeNormalizedReplacer**: Handles escaped characters
7. **TrimmedBoundaryReplacer**: Flexible boundary whitespace
8. **ContextAwareReplacer**: Uses surrounding context
9. **MultiOccurrenceReplacer**: Finds all exact matches

### Example: Block Anchor Matching

```typescript
// Works even if middle lines have slight differences
oldString = `
function process() {
  // some code here
  return result;
}
`

// Matches based on:
// - First line: "function process() {"
// - Last line: "}"
// - Middle content similarity
```

### Levenshtein Distance for Fuzzy Matching

```typescript
// From: packages/opencode/src/tool/edit.ts
function levenshtein(a: string, b: string): number {
  // Calculates edit distance between strings
  // Used to find "close enough" matches
}

// Thresholds:
const SINGLE_CANDIDATE_SIMILARITY_THRESHOLD = 0.0    // Very lenient
const MULTIPLE_CANDIDATES_SIMILARITY_THRESHOLD = 0.3  // More strict
```

### Why Multiple Strategies?

The LLM might:
- Copy text with different indentation
- Include/exclude trailing whitespace
- Have slight spacing variations

The multi-strategy approach handles these gracefully.

### Error Handling

```typescript
// If oldString not found:
throw new Error("oldString not found in content")

// If multiple matches found:
throw new Error(
  "Found multiple matches. Provide more surrounding lines " +
  "in oldString to identify the correct match."
)
```

### Diff Generation

After making changes, OpenCode generates a unified diff:

```typescript
import { createTwoFilesPatch } from "diff"

const diff = createTwoFilesPatch(
  filePath, filePath,
  contentOld, contentNew
)
```

**Example Output:**
```diff
--- src/utils.ts
+++ src/utils.ts
@@ -42,7 +42,7 @@
 function calculate(x) {
-  return x * 2;
+  return x * 3;
 }
```

This diff is:
- Shown to user for approval
- Used for diagnostics
- Tracked in session metadata

---

## 2. Patch Tool - Multi-File Changes

### How It Works

The `patch` tool applies changes to multiple files at once:

```typescript
// From: packages/opencode/src/tool/patch.ts

patch({
  patchText: `
*** Add File: src/newfile.ts
<file contents>

*** Update File: src/existing.ts
<<<<<<<
old content
=======
new content
>>>>>>>

*** Delete File: src/oldfile.ts
  `
})
```

### Patch Format

OpenCode uses a custom patch format:

#### Adding Files
```
*** Add File: path/to/new/file.ts
export const greeting = "Hello, world!";
```

#### Updating Files
```
*** Update File: path/to/existing/file.ts
<<<<<<<
function old() {
  return 1;
}
=======
function new() {
  return 2;
}
>>>>>>>
```

#### Deleting Files
```
*** Delete File: path/to/file.ts
```

#### Moving Files
```
*** Update File: old/path/file.ts
*** Move to: new/path/file.ts
<<<<<<<
old content
=======
new content (optional modifications)
>>>>>>>
```

### Patch Parsing

```typescript
// From: packages/opencode/src/patch/index.ts

interface Hunk {
  type: "add" | "update" | "delete"
  path: string
  contents?: string          // for add
  move_path?: string         // for move
  chunks?: UpdateFileChunk[] // for update
}

interface UpdateFileChunk {
  old_lines: string[]
  new_lines: string[]
  change_context?: string
  is_end_of_file?: boolean
}
```

### Context-Aware Updates

The patch system can use context to locate changes:

```typescript
{
  old_lines: ["const x = 5;", "const y = 10;"],
  new_lines: ["const x = 7;", "const y = 12;"],
  change_context: "inside function calculate()"
}
```

### Safety Features

1. **Atomic Operations**: All changes validated before applying
2. **Permission Checks**: User approval required
3. **File Locking**: Prevents race conditions
4. **Rollback Support**: Can revert changes if needed

---

## 3. Multi-Edit Tool - Batch String Replacements

### How It Works

Make multiple edits to the same file in sequence:

```typescript
// From: packages/opencode/src/tool/multiedit.ts

multiedit({
  filePath: "/path/to/file.ts",
  edits: [
    {
      oldString: "const x = 5;",
      newString: "const x = 10;"
    },
    {
      oldString: "const y = 3;",
      newString: "const y = 6;"
    }
  ]
})
```

### Why Use Multi-Edit?

**Single Tool Call vs Multiple:**
```typescript
// Option 1: Multiple edit() calls
edit({ oldString: "x = 5", newString: "x = 10" })
edit({ oldString: "y = 3", newString: "y = 6" })
// Problem: File state changes between calls
// Problem: Second edit might fail if file changed

// Option 2: One multiedit() call
multiedit({
  edits: [
    { oldString: "x = 5", newString: "x = 10" },
    { oldString: "y = 3", newString: "y = 6" }
  ]
})
// Benefit: All changes applied atomically
// Benefit: File locked during entire operation
```

---

## 4. Write Tool - Creating New Files

For brand new files:

```typescript
write({
  filePath: "/path/to/new/file.ts",
  content: "export const greeting = 'Hello';"
})
```

**When to Use:**
- Creating completely new files
- Not modifying existing files

---

## The Diff Workflow in Practice

### Example: Adding Error Handling

**Step 1: Read the current code**
```typescript
read({ filePath: "src/api/handler.ts", offset: 50, limit: 30 })
```

**Returns:**
```typescript
50. async function handleRequest(req) {
51.   const data = await fetchData();
52.   return processData(data);
53. }
```

**Step 2: Apply the change**
```typescript
edit({
  filePath: "src/api/handler.ts",
  oldString: `async function handleRequest(req) {
  const data = await fetchData();
  return processData(data);
}`,
  newString: `async function handleRequest(req) {
  try {
    const data = await fetchData();
    return processData(data);
  } catch (error) {
    console.error("Request failed:", error);
    throw error;
  }
}`
})
```

**Token Analysis:**
- Reading: ~30 lines = ~120 tokens
- Edit oldString: ~4 lines = ~40 tokens
- Edit newString: ~8 lines = ~80 tokens
- **Total: ~240 tokens**

**vs. Full File Approach:**
- Would need to output entire file: 500+ lines = ~2000 tokens
- **Savings: ~1760 tokens (88% reduction)**

---

## Advanced Features

### 1. File Time Tracking

OpenCode tracks when files are read/written:

```typescript
// From: packages/opencode/src/file/time.ts

// Prevents editing files that changed since last read
await FileTime.assert(sessionID, filePath)

// Updates tracking after changes
FileTime.read(sessionID, filePath)
```

**Why This Matters:**
- Prevents conflicting edits
- Ensures LLM is working with current content
- Avoids "lost update" problems

### 2. LSP Integration

After each edit, OpenCode runs diagnostics:

```typescript
// From: packages/opencode/src/tool/edit.ts

await LSP.touchFile(filePath, true)
const diagnostics = await LSP.diagnostics()

// If errors found:
if (errors.length > 0) {
  return `This file has errors, please fix:
  ${errors.map(LSP.Diagnostic.pretty).join("\n")}`
}
```

This provides immediate feedback about syntax errors or type issues.

### 3. Diff Trimming

Generated diffs are cleaned up for better readability:

```typescript
// From: packages/opencode/src/tool/edit.ts

function trimDiff(diff: string): string {
  // Removes common leading indentation
  // Makes diffs more compact
  // Preserves meaningful whitespace differences
}
```

---

## Benefits of the Diff Strategy

### 1. Token Efficiency
- **90%+ token savings** on large file edits
- Only changed lines are sent/received
- Reduces API costs dramatically

### 2. Precision
- Changes are surgical, not broad
- Less chance of unintended modifications
- Clear indication of what changed

### 3. Safety
- File locking prevents race conditions
- Time tracking prevents stale edits
- Atomic operations for multi-file changes

### 4. Debuggability
- Diffs clearly show what changed
- Easy to review changes
- Can revert specific changes

### 5. LLM-Friendly
- LLM doesn't need to regenerate entire files
- Reduces risk of truncation
- Faster generation times

---

## Common Patterns

### Pattern 1: Rename Variable
```typescript
edit({
  filePath: "src/utils.ts",
  oldString: "let userId = 123;",
  newString: "let userID = 123;",
  replaceAll: true  // Rename all occurrences
})
```

### Pattern 2: Add Import
```typescript
edit({
  filePath: "src/component.tsx",
  oldString: `import React from 'react';`,
  newString: `import React from 'react';
import { useState } from 'react';`
})
```

### Pattern 3: Wrap Existing Code
```typescript
edit({
  filePath: "src/api.ts",
  oldString: `const result = await fetch(url);`,
  newString: `try {
  const result = await fetch(url);
} catch (error) {
  handleError(error);
}`
})
```

---

## Error Recovery

### If Edit Fails

```typescript
// Error: "oldString not found"
// Solution: Read file again to get current content
read({ filePath: "src/file.ts" })

// Error: "Multiple matches found"
// Solution: Include more context
edit({
  oldString: `
  function helper() {
    // include more surrounding lines
    const x = 5;
    // to make match unique
  }
  `,
  newString: "..."
})
```

---

## Performance Characteristics

### Edit Tool
- **Time**: O(n) where n = file size
- **Space**: O(n) for diff generation
- **Token Usage**: O(change_size) - minimal

### Patch Tool
- **Time**: O(n × m) where n = files, m = file size
- **Space**: O(n × m)
- **Token Usage**: O(total_changes) - can batch multiple files

### Multi-Edit Tool
- **Time**: O(n × e) where n = file size, e = number of edits
- **Space**: O(n)
- **Token Usage**: O(total_changes)

---

## Key Takeaways

1. **Always use diff strategy** - Never output entire files unless necessary
2. **Read before editing** - Ensure you have current content
3. **Provide enough context** - Make oldString unique
4. **Use appropriate tool** - edit for single change, multiedit for multiple, patch for multi-file
5. **Review diffs** - Generated diffs show exactly what changed
6. **Handle errors gracefully** - Re-read file if edit fails

The diff strategy is fundamental to OpenCode's efficiency and precision. It allows working with large codebases while using minimal tokens and making reliable, targeted changes.

---

## See Also

- [Context Management Documentation](./01-context-management.md) - How OpenCode manages context limits
- [Tool Reference](./03-tool-reference.md) - Complete tool documentation
