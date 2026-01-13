# Visual Guide: OpenCode Context & Diff Strategy

This visual guide provides diagrams and flowcharts to help understand OpenCode's architecture.

---

## Context Management Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER REQUESTS TASK                           │
│                  "Add error handling to payment"                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────▼────────────────┐
         │   Should I search or read?      │
         │   Context: 5K/200K tokens       │
         └───────────────┬────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
        ┌───▼────┐              ┌────▼────┐
        │ GREP   │              │  GLOB   │
        │Pattern │              │ Pattern │
        │Search  │              │ Search  │
        └───┬────┘              └────┬────┘
            │                         │
            └────────────┬────────────┘
                         │
                 ┌───────▼────────┐
                 │  Found Files   │
                 │  src/pay.ts:42 │
                 └───────┬────────┘
                         │
                 ┌───────▼────────────────┐
                 │   READ (targeted)      │
                 │   offset: 35           │
                 │   limit: 50            │
                 │   = 50 lines (~200 tok)│
                 └───────┬────────────────┘
                         │
                 ┌───────▼────────────────┐
                 │   EDIT (diff)          │
                 │   oldString + newString│
                 │   = ~80 tokens         │
                 └───────┬────────────────┘
                         │
                 ┌───────▼────────────────┐
                 │   LSP VALIDATION       │
                 │   Check for errors     │
                 └───────┬────────────────┘
                         │
                 ┌───────▼────────────────┐
                 │   SUCCESS!             │
                 │   Total: ~380 tokens   │
                 │   vs 2000+ naive       │
                 └────────────────────────┘

SAVINGS: 80%+ tokens
```

---

## Token Budget Breakdown

```
┌──────────────────────────────────────────────────────────────────┐
│                 LLM CONTEXT WINDOW (200K tokens)                  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ SYSTEM PROMPT (cached) - 1,500 tokens                       ││
│  │ - Core instructions                                          ││
│  │ - Agent personality                                          ││
│  │ - Tool descriptions                                          ││
│  │ Cost: $0.30/M (90% discount from caching)                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ CONVERSATION HISTORY - 50K-150K tokens                      ││
│  │ - User messages                                              ││
│  │ - Assistant responses                                        ││
│  │ - Tool call results                                          ││
│  │                                                              ││
│  │ When full: COMPACTION ──────┐                               ││
│  │  1. Prune old outputs       │ Saves 20K-40K                ││
│  │  2. Summarize if needed     │ Condense to 10K              ││
│  └─────────────────────────────┴──────────────────────────────┘│
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ OUTPUT BUFFER (reserved) - 8K-32K tokens                    ││
│  │ - Space for LLM response                                     ││
│  │ - Tool call parameters                                       ││
│  │ - Reasoning (if supported)                                   ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                   │
│  USABLE = 200K - 32K = 168K tokens                               │
│  Trigger compaction at: ~150K tokens (90% full)                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Diff Strategy: Edit Tool Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDIT TOOL EXECUTION                           │
└─────────────────────────────────────────────────────────────────┘

Input Parameters:
┌──────────────────────────────────────┐
│ filePath: "src/api/handler.ts"      │
│ oldString: "const x = 5;"            │
│ newString: "const x = 10;"           │
└────────────┬─────────────────────────┘
             │
    ┌────────▼──────────┐
    │ File Lock Acquired│
    └────────┬──────────┘
             │
    ┌────────▼──────────────────┐
    │ Read Current File Content │
    └────────┬──────────────────┘
             │
    ┌────────▼───────────────────────────────────────┐
    │ Smart Matching (9 strategies in sequence)      │
    │                                                │
    │ 1. ✓ Exact match                              │
    │ 2. ✓ Line-trimmed                             │
    │ 3. ✓ Block anchor (Levenshtein)              │
    │ 4. ✓ Whitespace normalized                    │
    │ 5. ✓ Indentation flexible                     │
    │ 6. ✓ Escape normalized                        │
    │ 7. ✓ Trimmed boundary                         │
    │ 8. ✓ Context aware                            │
    │ 9. ✓ Multi-occurrence                         │
    │                                                │
    │ → Found unique match!                          │
    └────────┬───────────────────────────────────────┘
             │
    ┌────────▼─────────────┐
    │ Apply Replacement    │
    │ newContent = ...     │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Generate Diff        │
    │ (unified format)     │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Ask User Permission  │
    │ (show diff)          │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Write to File        │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Run LSP Diagnostics  │
    │ Check for errors     │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Update File Tracking │
    └────────┬─────────────┘
             │
    ┌────────▼─────────────┐
    │ Return Result        │
    │ - Diff shown         │
    │ - Errors if any      │
    └──────────────────────┘
```

---

## File Time Tracking

```
SESSION TIMELINE
═════════════════════════════════════════════════════════════

Time 0: Session starts
  │
  ├─ read("file.ts")
  │    └─ FileTime.read(session, "file.ts", timestamp: 1000)
  │       Record: session_abc → file.ts → 1000
  │
Time 1: Ready to edit
  │
  ├─ edit("file.ts", ...)
  │    ├─ FileTime.assert(session, "file.ts")
  │    │    └─ Check: file mtime (1000) == recorded (1000) ✓
  │    │
  │    ├─ Apply changes...
  │    │
  │    └─ FileTime.read(session, "file.ts", timestamp: 2000)
  │         Update: session_abc → file.ts → 2000
  │
Time 2: External change happens
  │  (e.g., git pull, IDE save)
  │  File mtime now: 2500
  │
Time 3: Try to edit again
  │
  └─ edit("file.ts", ...)
       └─ FileTime.assert(session, "file.ts")
            └─ Check: file mtime (2500) != recorded (2000) ✗
                ERROR: "File modified since last read"
                SOLUTION: Must read file again

PREVENTS: Lost updates, conflicting edits
ENABLES: Safe concurrent development
```

---

## Tool Selection Decision Tree

```
                     ┌─────────────────┐
                     │  What do I need? │
                     └────────┬─────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼──────┐     ┌─────▼──────┐     ┌─────▼──────┐
    │Find by name│     │Find in code│     │  Examine   │
    └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
          │                   │                   │
          │                   │                   │
      ┌───▼───┐           ┌──▼───┐           ┌──▼───┐
      │ glob  │           │ grep │           │ read │
      │"*.ts" │           │"func"│           │file  │
      └───────┘           └──────┘           └──────┘
                                                  │
                          ┌───────────────────────┤
                          │                       │
                    ┌─────▼──────┐         ┌─────▼──────┐
                    │  Navigate  │         │   Modify   │
                    └─────┬──────┘         └─────┬──────┘
                          │                       │
                    ┌─────▼──────┐         ┌─────┴─────┬──────────┐
                    │    LSP     │         │           │          │
                    │ Go to def  │       ┌─▼──┐    ┌───▼───┐  ┌──▼──┐
                    │ Find refs  │       │edit│    │multiedit│  │patch│
                    │ Symbols    │       │ 1  │    │ same   │  │multi│
                    └────────────┘       │file│    │ file   │  │file │
                                         └────┘    └────────┘  └─────┘

Decision factors:
- Speed: glob > grep > read
- Precision: read > grep > glob  
- Token cost: LSP (0) < grep < read < edit
```

---

## Compaction Process

```
CONTEXT WINDOW FILLING UP
═════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────┐
│ BEFORE COMPACTION                                         │
│                                                           │
│ Token usage: 170K / 200K (85% full)                      │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ System prompt (1.5K)                                 │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 1: User + Assistant (5K)                       │ │
│ │   - grep results                                     │ │
│ │   - read output                                      │ │
│ │   - edit diff                                        │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 2: User + Assistant (20K) ← OLD                │ │
│ │   - Large bash output                                │ │
│ │   - Multiple file reads                              │ │
│ │   - Several edits                                    │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 3: User + Assistant (30K) ← OLD                │ │
│ │   - Complex refactoring                              │ │
│ │   - Many tool calls                                  │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 4: User + Assistant (15K)                      │ │
│ │   - Recent work                                      │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 5: Current work in progress...                 │ │
│ └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

                        ↓ COMPACTION ↓

┌──────────────────────────────────────────────────────────┐
│ AFTER COMPACTION                                          │
│                                                           │
│ Token usage: 60K / 200K (30% full) ✓                     │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ System prompt (1.5K)                                 │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ SUMMARY MESSAGE (10K) ← NEW                         │ │
│ │                                                      │ │
│ │ "We've been working on adding error handling to     │ │
│ │  the payment system. So far we've:                  │ │
│ │  - Modified src/api/payment.ts to add try-catch     │ │
│ │  - Updated src/types.ts with error types            │ │
│ │  - Added logging in src/logger.ts                   │ │
│ │  Next steps: Add integration tests"                 │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 4: User + Assistant (15K)                      │ │
│ │   - Kept (recent)                                    │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Turn 5: Current work in progress...                 │ │
│ └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

RESULT: Saved 110K tokens (65% reduction)
       Can continue for many more turns!
```

---

## Token Cost Comparison

### Scenario: Edit Function in 1000-Line File

```
┌─────────────────────────────────────────────────────────────┐
│ APPROACH 1: FULL FILE REPLACEMENT                           │
└─────────────────────────────────────────────────────────────┘

Step 1: Read entire file
  Input: Read request (50 tokens)
  Output: 1000 lines (~4000 tokens) ← EXPENSIVE
  
Step 2: LLM generates entire file
  Input: 4000 tokens (file content)
  Output: 1000 lines (~4000 tokens) ← VERY EXPENSIVE

TOTAL: ~8,100 tokens
COST (Claude 3.5): $0.16
TIME: 60-90 seconds (slow generation)


┌─────────────────────────────────────────────────────────────┐
│ APPROACH 2: DIFF STRATEGY (OpenCode)                        │
└─────────────────────────────────────────────────────────────┘

Step 1: Search for function
  Input: Grep request (50 tokens)
  Output: Match location (50 tokens)
  
Step 2: Read relevant section
  Input: Read request (50 tokens)
  Output: 50 lines (~200 tokens)
  
Step 3: Generate diff
  Input: 200 tokens (section)
  Output: 20 lines (~80 tokens) ← EFFICIENT

TOTAL: ~430 tokens
COST (Claude 3.5): $0.009
TIME: 5-10 seconds (fast)

────────────────────────────────────────────────────────────────
SAVINGS: 95% tokens, 94% cost, 10x faster ✓
────────────────────────────────────────────────────────────────
```

---

## Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         OPENCODE SYSTEM                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    USER INTERFACES (Frontend)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   TUI    │  │   Web    │  │ Desktop  │  │   API    │       │
│  │(Terminal)│  │   App    │  │   App    │  │  Client  │       │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└────────┼────────────┼─────────────┼─────────────┼──────────────┘
         │            │             │             │
         └────────────┴─────────────┴─────────────┘
                      │
┌─────────────────────▼─────────────────────────────────────────┐
│                    SESSION LAYER                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Session Manager                                           │ │
│  │ - Message history                                         │ │
│  │ - Token tracking                                          │ │
│  │ - Compaction control                                      │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────┬───────────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────────┐
│                    LLM INTEGRATION                             │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Provider Abstraction                                      │ │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │ │
│  │ │ Anthropic│ │  OpenAI  │ │  Google  │ │  Local   │    │ │
│  │ │  Claude  │ │   GPT    │ │  Gemini  │ │  Models  │    │ │
│  │ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────┬───────────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────────┐
│                    TOOL SYSTEM                                 │
│  ┌────────────┬────────────┬────────────┬──────────────┐     │
│  │File Ops    │Code Search │Language    │Other         │     │
│  │            │            │Intelligence│              │     │
│  │┌──────────┐│┌──────────┐│┌──────────┐│┌────────────┐│    │
│  ││read      │││grep      │││LSP       │││bash        ││    │
│  ││write     │││glob      │││go to def │││task        ││    │
│  ││edit      │││codesearch│││find refs │││skill       ││    │
│  ││multiedit │││          │││diagnostics│││websearch  ││    │
│  ││patch     │││          │││          │││           ││    │
│  │└──────────┘│└──────────┘│└──────────┘│└────────────┘│    │
│  └────────────┴────────────┴────────────┴──────────────┘     │
└───────────────────────┬───────────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────────┐
│                CORE SERVICES                                   │
│  ┌──────────────────┬──────────────────┬──────────────────┐  │
│  │Context Mgmt      │File System       │Safety            │  │
│  │                  │                  │                  │  │
│  │- Token estimate  │- Time tracking   │- Permissions     │  │
│  │- Truncation      │- Snapshots       │- Locking         │  │
│  │- Compaction      │- Git integration │- Validation      │  │
│  │- Pruning         │- Change tracking │- Rollback        │  │
│  └──────────────────┴──────────────────┴──────────────────┘  │
└────────────────────────────────────────────────────────────────┘

DATA FLOW:
User input → Session → LLM → Tools → File System → Results → User

FEEDBACK LOOPS:
- Token tracking informs compaction
- File tracking prevents conflicts  
- LSP provides immediate validation
- Permissions gate dangerous operations
```

---

## Quick Decision Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                    QUICK DECISIONS                           │
└─────────────────────────────────────────────────────────────┘

SITUATION: "Need to find code"
├─ Know file name?
│  ├─ YES → Use GLOB
│  └─ NO → Use GREP
│
SITUATION: "Need to read code"
├─ File size known?
│  ├─ < 500 lines → READ entire file
│  └─ > 500 lines → READ with offset/limit
│
SITUATION: "Need to modify code"
├─ How many changes?
│  ├─ 1 location → Use EDIT
│  ├─ Multiple in same file → Use MULTIEDIT
│  └─ Multiple files → Use PATCH
│
SITUATION: "Token usage high"
├─ Check recent tool outputs
│  ├─ Large outputs? → Should be truncated
│  ├─ Re-reading files? → Unnecessary
│  └─ Need compaction? → Enable auto-compaction
│
SITUATION: "Edit failed"
├─ Error message says?
│  ├─ "not found" → Re-read file
│  ├─ "multiple matches" → Add context
│  └─ "file modified" → Read file again
│
SITUATION: "Exploring codebase"
├─ Start with:
│  1. LS (directory structure)
│  2. GREP (find patterns)
│  3. READ (examine matches)
│  └─ Never start with reading everything!

RULE OF THUMB:
Search → Verify → Read → Modify
(Each step costs fewer tokens than the next)
```

---

## Performance Optimization Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              OPTIMIZATION TECHNIQUES                         │
└─────────────────────────────────────────────────────────────┘

1. PARALLEL TOOL CALLS
   ┌──────────────────────────────────────┐
   │ Sequential (slow):                   │
   │ read(file1) → wait → read(file2)    │
   │ Total: 2x latency                    │
   └──────────────────────────────────────┘
   
   ┌──────────────────────────────────────┐
   │ Parallel (fast):                     │
   │ Promise.all([read(file1), read(file2)])│
   │ Total: 1x latency                    │
   └──────────────────────────────────────┘

2. PROMPT CACHING
   ┌──────────────────────────────────────┐
   │ First request:                       │
   │ System prompt: 1500 tokens × $3/M    │
   │ Cost: $0.0045                        │
   └──────────────────────────────────────┘
   
   ┌──────────────────────────────────────┐
   │ Subsequent requests:                 │
   │ Cached: 1500 tokens × $0.30/M       │
   │ Cost: $0.00045 (90% discount!)      │
   └──────────────────────────────────────┘

3. STREAMING RESPONSES
   ┌──────────────────────────────────────┐
   │ Without streaming:                   │
   │ Wait... → Get entire response        │
   │ User sees nothing until complete     │
   └──────────────────────────────────────┘
   
   ┌──────────────────────────────────────┐
   │ With streaming:                      │
   │ Text appears → Tool executes →       │
   │ User sees progress immediately       │
   └──────────────────────────────────────┘

4. LSP BACKGROUND PROCESSING
   ┌──────────────────────────────────────┐
   │ Blocking:                            │
   │ Edit → Wait for LSP → Show result   │
   └──────────────────────────────────────┘
   
   ┌──────────────────────────────────────┐
   │ Background:                          │
   │ Edit → Show success → LSP runs async │
   │ Errors shown when ready              │
   └──────────────────────────────────────┘
```

---

**See detailed documentation:**
- [Architecture Overview](./00-architecture-overview.md)
- [Context Management](./01-context-management.md)
- [Diff Strategy](./02-diff-strategy.md)
- [Quick Reference](./99-quick-reference.md)
