# OpenCode Documentation

Welcome to the OpenCode internal architecture documentation! This folder contains detailed explanations of how OpenCode works under the hood.

## 📚 Documentation Index

### [VISUAL-GUIDE.md](./VISUAL-GUIDE.md) 📊
**Visual Guide with Diagrams** - ASCII diagrams and flowcharts
- Context management flow diagram
- Token budget breakdown visualization
- Edit tool execution flow
- File time tracking timeline
- Tool selection decision tree
- Compaction process before/after
- Complete system architecture diagram

### [99-quick-reference.md](./99-quick-reference.md) ⚡
**Quick Reference Guide** - Fast lookup for common patterns and workflows
- Context management in 5 steps
- Diff strategy in 3 rules
- Token cost comparisons
- Common workflows and error handling
- Tool selection guide
- Performance tips

### [00-architecture-overview.md](./00-architecture-overview.md)
**Complete architecture overview** - Start here to understand OpenCode's overall design
- High-level component diagram
- Data flow examples
- Core systems explained
- Design principles

### [01-context-management.md](./01-context-management.md)
**Context Management: Reading Code Without Exceeding Limits**
- The challenge of large codebases and token limits
- Selective file reading with pagination
- Smart search strategies (grep, glob, LSP)
- Truncation and overflow handling
- Context compaction and pruning
- Real-world examples with token counts

### [02-diff-strategy.md](./02-diff-strategy.md)
**Diff Strategy: Applying Code Changes Efficiently**
- Why diff strategy beats full file replacement
- The edit tool and smart matching algorithms
- Patch tool for multi-file changes
- Error handling and recovery
- Token efficiency comparisons
- Common patterns and best practices

---

## 🎯 Quick Start

**Need visual diagrams?** Check out:
- [Visual Guide](./VISUAL-GUIDE.md) - Flowcharts and ASCII diagrams

**Need a quick reference?** Start with:
- [Quick Reference Guide](./99-quick-reference.md) - Fast lookup for common patterns, workflows, and token costs

**New to OpenCode?** Read in this order:
1. [Architecture Overview](./00-architecture-overview.md) - Get the big picture
2. [Context Management](./01-context-management.md) - Understand how code is read
3. [Diff Strategy](./02-diff-strategy.md) - Learn how changes are applied

**Looking for something specific?**
- Diagrams and flowcharts → [Visual Guide](./VISUAL-GUIDE.md)
- Token management → [Context Management](./01-context-management.md)
- Edit/patch tools → [Diff Strategy](./02-diff-strategy.md)
- Complete system flow → [Architecture Overview](./00-architecture-overview.md)
- Quick patterns → [Quick Reference](./99-quick-reference.md)

---

## 🎓 For Different Audiences

### For Beginners
These docs are written to be accessible to developers who are new to:
- AI coding agents
- Context window management
- Diff/patch strategies

No prior knowledge assumed! Each concept is explained from first principles.

### For Contributors
Understanding these internals will help you:
- Add new tools effectively
- Improve context management strategies
- Optimize token usage
- Debug issues in production
- Design new features that fit the architecture

### For Advanced Users
Learn how to:
- Configure compaction behavior
- Create custom agents
- Write plugins that hook into the tool system
- Optimize for your specific use case

---

## 💡 Key Concepts

### 1. Context Window Management
OpenCode works with LLMs that have limited context windows (typically 128K-200K tokens). To work with large codebases:
- **Selective Reading**: Read only relevant files/sections
- **Search First**: Use grep/glob before reading
- **Pagination**: Read files in chunks
- **Truncation**: Show previews for large outputs
- **Compaction**: Summarize old conversation history

→ Details in [Context Management](./01-context-management.md)

### 2. Diff/Patch Strategy
Instead of outputting entire files, OpenCode uses diffs:
- **Edit Tool**: Surgical string replacements
- **Smart Matching**: 9 strategies to find code even with formatting differences
- **Patch Tool**: Multi-file atomic changes
- **Token Efficient**: 90%+ savings vs full rewrites

→ Details in [Diff Strategy](./02-diff-strategy.md)

### 3. Tool System
OpenCode provides the LLM with tools to:
- Read and edit files
- Search code (grep, glob)
- Execute shell commands
- Access language intelligence (LSP)
- Delegate to sub-agents

→ Overview in [Architecture](./00-architecture-overview.md#3-tool-system)

### 4. Safety & Reliability
- **File Locking**: Prevents concurrent edits
- **Time Tracking**: Detects stale edits
- **Snapshots**: Git-based rollback capability
- **Permissions**: Control what agents can do
- **LSP Validation**: Immediate error detection

→ Details in [Architecture](./00-architecture-overview.md#6-file-tracking-system)

---

## 📊 Example: Complete Workflow

Here's how OpenCode handles "Add error handling to payment function":

```
1. User Request
   ↓
2. LLM decides to search
   ↓ grep({ pattern: "function.*payment" })
   ↓ Result: "src/payment/processor.ts:42"
   ↓
3. LLM reads relevant section
   ↓ read({ filePath: "...", offset: 35, limit: 50 })
   ↓ Returns: 50 lines around the function
   ↓
4. LLM generates edit
   ↓ edit({ oldString: "...", newString: "... try/catch ..." })
   ↓ Smart matching finds exact location
   ↓ Generates diff for approval
   ↓
5. User approves → Change applied
   ↓
6. LSP validates → No errors
   ↓
7. Success!

Token Usage:
- Grep: ~100 tokens
- Read: ~600 tokens  
- Edit: ~200 tokens
Total: ~900 tokens (vs ~5000 for reading entire file)
```

→ Full walkthrough in [Architecture](./00-architecture-overview.md#data-flow-complete-example)

---

## 🔧 Technical Details

### Token Estimation
```typescript
// Rough approximation
1 token ≈ 4 characters

// Example:
"const x = 5;" → ~12 chars → ~3 tokens
```

### Context Limits
```typescript
// Typical models
GPT-4 Turbo: 128K tokens
Claude 3.5 Sonnet: 200K tokens
Gemini 1.5 Pro: 1M tokens (but costly)

// OpenCode reserves space for output
usable_context = total_context - output_buffer
output_buffer = 8K-32K tokens (configurable)
```

### Compaction Triggers
```typescript
// Compaction triggered when:
token_usage > (context_limit - output_buffer)

// Process:
1. Try pruning (remove old tool outputs)
   → Saves 20K-40K tokens typically
2. If still over limit, create summary
   → Replaces history with concise context
```

---

## 🚀 Performance Characteristics

### Edit Operations
- **Time**: O(n) where n = file size
- **Space**: O(n) for diff generation  
- **Tokens**: O(change_size) - proportional to change, not file size

### Context Management
- **Pruning**: Removes 20K-40K tokens (fast)
- **Compaction**: Creates summary (1 LLM call, ~30 seconds)
- **Frequency**: As needed when approaching limit

### Search Operations
- **Grep**: Fast (uses ripgrep), returns matches only
- **LSP**: Near-instant (cached symbol index)
- **Read**: Fast for chunks, slower for full large files

---

## 📝 Common Patterns

### Pattern: Find → Read → Edit
```typescript
// 1. Find the file
grep({ pattern: "class UserService" })

// 2. Read relevant section  
read({ filePath: "src/user.ts", offset: 100, limit: 50 })

// 3. Make precise change
edit({ 
  oldString: "...",
  newString: "..." 
})
```

### Pattern: Multi-file Refactor
```typescript
// Use patch tool for atomic multi-file changes
patch({
  patchText: `
*** Update File: src/types.ts
<<<<<<<
export type User = {...}
=======
export interface User {...}
>>>>>>>

*** Update File: src/api.ts
<<<<<<<
import type { User } from './types'
=======
import { User } from './types'
>>>>>>>
  `
})
```

### Pattern: Explore Then Act
```typescript
// 1. List structure
ls({ path: "src/api", depth: 2 })

// 2. Search for patterns
grep({ pattern: "TODO", path: "src/api" })

// 3. Read specific files
read({ filePath: "src/api/handler.ts" })

// 4. Make informed changes
edit({ ... })
```

---

## 🤔 FAQ

### Q: Why use diff strategy instead of full file replacement?
**A:** Token efficiency and precision. Editing 10 lines in a 1000-line file:
- Diff strategy: ~80 tokens output
- Full replacement: ~4000 tokens output
- **Savings: 98% fewer tokens!**

Plus, diff is more reliable (less chance of errors in unchanged code).

### Q: What happens when context window fills up?
**A:** OpenCode automatically:
1. Prunes old tool call outputs (saves 20K-40K tokens)
2. If still full, creates a summary of conversation
3. Summary becomes new conversation starting point

See [Context Management](./01-context-management.md#5-context-compaction-message-history-pruning)

### Q: How does OpenCode avoid reading unnecessary files?
**A:** Multi-stage approach:
1. Use `grep` to search without reading
2. Use `glob` to find files by name
3. Use LSP to navigate code structure
4. Only `read` confirmed relevant files
5. Use `offset`/`limit` to read sections, not entire files

See [Context Management](./01-context-management.md#2-smart-search-before-reading)

### Q: What if the edit tool can't find the code to replace?
**A:** The edit tool has 9 matching strategies to handle:
- Different indentation
- Extra/missing whitespace
- Escaped characters
- Slight variations

If still not found, error message guides LLM to:
- Re-read file for current content
- Provide more context in `oldString`

See [Diff Strategy](./02-diff-strategy.md#smart-matching-system)

### Q: How does OpenCode prevent conflicting edits?
**A:** File time tracking:
```typescript
// OpenCode tracks when files are read
FileTime.read(sessionID, filePath)

// Before editing, verifies file unchanged
await FileTime.assert(sessionID, filePath)
// Throws error if file modified

// After editing, updates tracking
FileTime.read(sessionID, filePath)
```

See [Architecture](./00-architecture-overview.md#6-file-tracking-system)

---

## 🔍 Debugging Tips

### Excessive Token Usage?
1. Check if files are being re-read unnecessarily
2. Verify compaction is enabled
3. Look for large tool outputs (should be truncated)
4. Consider using grep before read

### Edit Failures?
1. Verify file was read in current session
2. Check if `oldString` matches exactly (including whitespace)
3. Try providing more context in `oldString`
4. Use `replaceAll: true` if appropriate

### Context Overflow?
1. Enable auto-compaction in config
2. Reduce initial read sizes
3. Use more targeted grep searches
4. Consider breaking task into smaller chunks

---

## 📦 Related Resources

### OpenCode Main Documentation
- [Main README](../README.md) - Installation and getting started
- [Contributing Guide](../CONTRIBUTING.md) - How to contribute
- [Agents Guide](../AGENTS.md) - Agent configuration

### Source Code References
- `packages/opencode/src/tool/` - Tool implementations
- `packages/opencode/src/session/` - Session management
- `packages/opencode/src/lsp/` - LSP integration
- `packages/opencode/src/patch/` - Patch system

### External Resources
- [Anthropic Claude Documentation](https://docs.anthropic.com/)
- [OpenAI API Documentation](https://platform.openai.com/docs)
- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)

---

## 📧 Questions or Feedback?

- **Issues**: [GitHub Issues](https://github.com/anomalyco/opencode/issues)
- **Discussions**: [GitHub Discussions](https://github.com/anomalyco/opencode/discussions)
- **Discord**: [OpenCode Community](https://opencode.ai/discord)

---

## ✨ Contributing to Docs

Found an error or want to improve these docs?

1. Docs are in `/docs` folder
2. Written in Markdown
3. Follow existing structure and style
4. Submit PR with clear description

We appreciate contributions that make OpenCode more understandable!

---

**Last Updated**: January 2026  
**OpenCode Version**: 1.x

Happy coding! 🚀
