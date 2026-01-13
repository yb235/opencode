# Quick Reference: OpenCode Context & Diff Strategy

This is a condensed quick reference guide. For detailed explanations, see the full documentation.

---

## Context Management in 5 Steps

### 1. Never Read Everything
```typescript
// ❌ Don't do this
read({ filePath: "large-file.ts" })  // Reads all 2000 lines

// ✅ Do this instead
grep({ pattern: "function target" })   // Find location first
read({ filePath: "file.ts", offset: 100, limit: 50 })  // Read section
```

### 2. Search Before Reading
```typescript
// Step 1: Find files
glob({ pattern: "**/*handler*.ts" })

// Step 2: Search content
grep({ pattern: "export.*Handler", include: "*.ts" })

// Step 3: Read confirmed matches
read({ filePath: "src/api/handler.ts" })
```

### 3. Use Pagination
```typescript
// For large files
read({ filePath: "file.ts", offset: 0, limit: 500 })     // First 500 lines
read({ filePath: "file.ts", offset: 500, limit: 500 })   // Next 500 lines
```

### 4. Let Truncation Work
When output exceeds limits:
- Preview shown (first/last N lines)
- Full output saved to temp file
- Use grep/read to explore saved file

### 5. Trust Compaction
When context fills:
- Old tool outputs pruned (saves ~30K tokens)
- If still full, conversation summarized
- Continue seamlessly with new context

---

## Diff Strategy in 3 Rules

### Rule 1: Always Use Diff, Never Full Replace
```typescript
// ❌ Bad: Output entire file (4000 tokens)
write({ filePath: "file.ts", content: entireFile })

// ✅ Good: Output only changes (80 tokens)
edit({ 
  filePath: "file.ts",
  oldString: "const x = 5;",
  newString: "const x = 10;"
})
```

**Savings: 98% fewer tokens**

### Rule 2: Read Before Editing
```typescript
// Step 1: Always read first
read({ filePath: "file.ts" })

// Step 2: Then edit
edit({ 
  oldString: "// exact match from read output",
  newString: "// new code"
})
```

### Rule 3: Provide Context for Uniqueness
```typescript
// ❌ If this appears multiple times
oldString: "return true;"

// ✅ Include surrounding context
oldString: `
function validate() {
  if (condition) {
    return true;
  }
}
`
```

---

## Token Cost Comparison

### Scenario: Edit 10 lines in 1000-line file

| Approach | Tokens | Cost (Claude) | Time |
|----------|--------|---------------|------|
| Full replace | ~4000 | $0.06 | Slow |
| Diff strategy | ~80 | $0.001 | Fast |
| **Savings** | **98%** | **98%** | **10x** |

### Scenario: Find and edit function

| Step | Tool | Tokens | Cumulative |
|------|------|--------|------------|
| Find | grep | ~100 | 100 |
| Read | read (50 lines) | ~200 | 300 |
| Edit | edit | ~80 | 380 |
| **Total** | | | **~400** |

vs. reading entire file: ~2000 tokens → **80% savings**

---

## Common Workflows

### Workflow 1: Add Feature
```typescript
// 1. Find relevant file
grep({ pattern: "class UserService" })
// → src/user/service.ts:15

// 2. Read relevant section
read({ filePath: "src/user/service.ts", offset: 10, limit: 50 })

// 3. Add new method
edit({
  oldString: `
  updateUser(id: string, data: User) {
    // ...
  }
}`, // End of class
  newString: `
  updateUser(id: string, data: User) {
    // ...
  }

  deleteUser(id: string) {
    return this.db.delete(id);
  }
}` // Added new method
})
```

### Workflow 2: Fix Bug Across Files
```typescript
// 1. Find all occurrences
grep({ pattern: "oldFunction", include: "*.ts" })

// 2. Read each file
read({ filePath: "file1.ts" })
read({ filePath: "file2.ts" })
read({ filePath: "file3.ts" })

// 3. Apply patch for atomic multi-file change
patch({
  patchText: `
*** Update File: file1.ts
<<<<<<<
oldFunction()
=======
newFunction()
>>>>>>>

*** Update File: file2.ts
<<<<<<<
oldFunction()
=======
newFunction()
>>>>>>>
  `
})
```

### Workflow 3: Explore Codebase
```typescript
// 1. See structure
ls({ path: "src", depth: 2 })

// 2. Search for patterns
grep({ pattern: "TODO|FIXME", path: "src" })

// 3. Read interesting files
read({ filePath: "src/module/file.ts" })

// No edits needed - stayed read-only
```

---

## Error Handling Quick Guide

### Error: "oldString not found"
**Cause**: File changed since read, or oldString doesn't match exactly
**Fix**: Re-read file to get current content

### Error: "Multiple matches found"
**Cause**: oldString appears in multiple places
**Fix**: Include more surrounding context in oldString

### Error: "File modified since last read"
**Cause**: File changed by external process or another tool
**Fix**: Re-read file before editing

### Context Overflow
**Cause**: Too many tokens in conversation
**Fix**: Enable auto-compaction in config (usually automatic)

---

## Tool Selection Guide

| Need | Tool | Why |
|------|------|-----|
| Find files by name | `glob` | Fast, pattern-based |
| Search code content | `grep` | Fast, doesn't read full files |
| Read file | `read` | Paginated, efficient |
| Create file | `write` | For new files only |
| Edit one place | `edit` | Surgical, 9 matching strategies |
| Edit multiple places | `multiedit` | Atomic, same file |
| Edit multiple files | `patch` | Atomic, multi-file |
| Navigate code | LSP tools | Instant, no reading needed |
| Run commands | `bash` | For builds, tests, etc. |

---

## Configuration Tips

### Enable Auto-Compaction
```json
// ~/.opencode/config.json
{
  "compaction": {
    "auto": true,
    "prune": true
  }
}
```

### Customize Token Limits
```json
{
  "tool": {
    "read": {
      "defaultLimit": 1000,  // lines per read
      "maxBytes": 30000      // 30KB max
    }
  }
}
```

---

## Performance Tips

### Tip 1: Parallel Reads
```typescript
// Call multiple reads in parallel
await Promise.all([
  read({ filePath: "file1.ts" }),
  read({ filePath: "file2.ts" }),
  read({ filePath: "file3.ts" })
])
```

### Tip 2: Use LSP for Navigation
```typescript
// Instead of reading entire file to find definition
// Use LSP (instant, no tokens)
LSP.definition(filePath, position)
```

### Tip 3: Cache System Prompts
System prompts are automatically cached, reducing costs by 90% on repeated requests.

### Tip 4: Batch Similar Operations
```typescript
// Instead of multiple edit() calls
multiedit({
  edits: [
    { oldString: "x = 1", newString: "x = 2" },
    { oldString: "y = 3", newString: "y = 4" }
  ]
})
```

---

## Debug Checklist

Excessive token usage?
- [ ] Files being re-read unnecessarily?
- [ ] Compaction enabled?
- [ ] Using grep before read?
- [ ] Reading full files instead of sections?

Edit failures?
- [ ] File read in current session?
- [ ] oldString matches exactly?
- [ ] Need more context in oldString?
- [ ] File modified externally?

Slow performance?
- [ ] Using LSP for navigation?
- [ ] Reading in parallel when possible?
- [ ] Truncation working properly?

---

## Key Metrics to Watch

### Token Usage
- **Target**: <5K tokens per turn for simple edits
- **Warning**: >20K tokens (review strategy)
- **Critical**: >50K tokens (compaction needed)

### Context Window
- **Safe**: <50% of limit
- **Warning**: >70% of limit
- **Critical**: >90% of limit (compaction triggered)

### Tool Calls per Turn
- **Optimal**: 2-4 tools (grep → read → edit)
- **Acceptable**: 5-8 tools
- **Excessive**: >10 tools (may need sub-agent)

---

## See Full Documentation

- [Architecture Overview](./00-architecture-overview.md)
- [Context Management](./01-context-management.md)
- [Diff Strategy](./02-diff-strategy.md)

---

**Remember**: 
- 🔍 Search before reading
- 📖 Read sections, not entire files
- ✏️ Edit with diffs, not full rewrites
- 🔄 Trust automatic compaction
- 📊 Monitor token usage

**Result**: Work with any codebase size, stay within limits, keep costs low! 🚀
