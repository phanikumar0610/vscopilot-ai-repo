# VS Code GitHub Copilot Context Window Analysis

**Date:** May 3, 2026  
**Relevance:** Angular Migration Skill v2 Implementation & Agent Coordination  
**Status:** Current as of May 2026

---

## Executive Summary

**Current State (May 2026):**
- VS Code Copilot has a 192k token context window with ~40% reserved for output
- Automatic context compaction via summarization when window fills up
- Manual compaction available via `/compact` command
- Large tool output written to disk instead of context

**Key Issue:**
Claude Opus 4.6 supports 1M context window, but GitHub Copilot caps it at 128K for economic/latency reasons

**Bottom Line:** Your Angular Migration Skill agents will likely hit context limits during long-running migrations. Built-in features help, but **proactive strategies are essential**.

---

## 1. Current Context Window Limits

### Available Context (As of May 2026)

| Model | Context Limit | Reserved Output | Practical Input | Status |
|-------|---|---|---|---|
| Claude Opus 4.6 | 192K tokens | ~60K (30%) | ~132K | Current (constrained from 1M) |
| Claude Sonnet 4.6 | 128K tokens | ~60K | ~68K | Current |
| GPT-4o | 128K tokens | ~40K | ~88K | Current |
| Gemini 2.5 Pro | 64K tokens | ~20K | ~44K | Current (constrained from 1M) |
| o4-mini | 100K tokens | ~30K | ~70K | Current |

**Critical Context:** GitHub hasn't updated limits despite Claude's 1M context becoming GA in early 2026, citing economic/latency constraints

### Your Situation with Angular Migration Skill

**Per-Agent Context Consumption:**
```
Orchestrator Agent Input:
  ├─ System instructions          ~2K tokens
  ├─ Migration plan JSON          ~5K tokens
  ├─ migration-state.json         ~3K tokens (grows per step)
  ├─ Reference docs (19→20)       ~8K tokens
  ├─ Execution log (grows)        ~10K tokens (per 5 steps)
  └─ Sub-agent outputs            ~5K tokens
  
  Total: ~33K tokens for initial setup + growing with each step
```

**With Multi-Step Migration (7 steps):**
- Initial: ~33K tokens
- After Step 2A: ~40K tokens
- After Step 2B: ~48K tokens
- After Step 3: ~55K tokens
- After Step 4: ~63K tokens (APPROACHING 50% of 128K)

**Conclusion:** Your migration will hit compaction triggers around Step 3-4.

---

## 2. Current Copilot Features for Context Management

### A. Automatic Context Compaction

**How It Works:**
When context window fills up, VS Code automatically compacts conversation by summarizing earlier messages transparently in background

**Trigger Point:** Compaction starts automatically at ~80% capacity with ~20% buffer for tool calls

**What Gets Lost:**
Compaction captures key points but fine-grained details like exact wording, full command outputs, and minor decisions may not be included

**Problem for Your Skill:**
If agent is in middle of Step 3 (test framework migration) and compaction happens, it might forget:
- Specific files modified in Step 2A
- Exact test failures encountered
- Dependency graph for remaining steps

### B. Manual Context Compaction (/compact)

**Command:** `/compact` in chat input, optionally with custom instructions (e.g., `/compact focus on database schema decisions`)

**Available Since:** February 2026 in VS Code Insiders

**Use Case:**
```
Before starting Step 3 (Test Framework):
/compact focus on Step 1 and Step 2A/2B results, keep dependency graph
```

**Problem for Your Skill:**
Agent must manually trigger this—no built-in automation for multi-step workflows.

### C. Checkpoints & Session Memory

**Feature:** Every compaction creates a checkpoint (numbered, titled file) with saved summary

**Benefit:** Allows resuming from checkpoint if session crashes

**Problem:** Doesn't prevent context loss, just allows recovery

### D. Large Output Handling

**Feature:** Large tool output written to disk instead of stuffed into context, so details aren't lost during compaction

**Benefit:** Test results, file diffs, logs don't consume context window

**Status:** Available as of February 2026

### E. Agent Memory & Plan Persistence

**Feature:** Agents share and store knowledge across Copilot coding agent, Copilot CLI, and code review; plans persist through compaction

**Benefit:** Agent can recall earlier decisions even after compaction

**Problem:** Only works for "plans," not execution details

---

## 3. Limitations & Known Issues

### Issue #1: Compaction Doesn't Reclaim Much Space

**Problem:** After compaction, context window still 50% full; manual `/compact` barely reduces it to 36%

**Root Cause:** Reserved output (~30% of context) + compaction overhead = limited space recovery

**Impact for Your Skill:**
- First compaction might free ~30K tokens
- Next compaction might only free ~10K tokens
- Eventually agent hits hard limit (~90K input available)

### Issue #2: Reserved Output Is Mandatory

**Problem:** ~30% of context reserved for output guarantee; not configurable, managed on backend

**Reason:** Prevents AI from running out of memory during long generations

**Impact:** You can never use full 128K for input; realistically ~90K available

### Issue #3: Compaction Loss Can Be Critical

**Problem:** When compaction happens automatically mid-task, agent might forget critical context

**Real Example:** Migration step 3 starts, compaction triggers, agent loses memory of which files were modified in step 2

### Issue #4: No Fine-Grained Control Over What's Preserved

**Problem:** User has no control over what agent retains/discards as context fills up

**Requested Feature:** `#remember` command to anchor critical info (as of Jan 2026, still requested, not implemented)

### Issue #5: Multi-Turn Context Grows Quickly

**Problem:** Each agent-human interaction adds tokens:
- Your prompt: +1-2K tokens
- Agent response: +2-5K tokens
- State updates: +1-2K tokens
- Tool results: +3-10K tokens

**7-step migration = 7-14 interactions = 20-70K tokens of conversation history**

---

## 4. Why Copilot Limits Are Lower Than Model Capability

**Technical Reasons:** Running 1M token inference at scale is expensive and significantly slower; interactive coding workflows are latency-sensitive

**Economic Reason:** Serving 1M contexts across large user base with sub-second response expectations isn't economically viable

**Practical Impact:** Copilot is optimized for quick interactions, not long-running agent workflows

---

## 5. Recommendations for Angular Migration Skill

### Recommendation 1: Use Explicit State Management (What You're Already Doing ✓)

**Strategy:** Separate `migration-state.json` from Copilot context

**Benefits:**
- State file lives on disk, not in context
- Orchestrator can reference state without consuming context
- Sub-agents load only relevant portions of state

**Implementation:**
```json
// migration-state.json (on disk, not in context)
{
  "steps_completed": ["1"],
  "steps_in_progress": ["2A", "2B"],
  "execution_log": [ /* only recent entries in context */ ]
}

// Each agent loads:
- Current step details only (~1-2K)
- Relevant reference docs only (~5K)
- Recent execution log only (~3K)
// Total: ~10K per agent run
```

**Why This Works:** You're treating file system as external memory, not relying on context

### Recommendation 2: Implement Session Checkpointing (CRITICAL)

**What:** Every 2-3 agent steps, create a checkpoint with:
- Completed steps summary
- Current state snapshot
- Key findings & decisions

**How:**
```
After Step 2B completes:
  /compact focus on Step 1, 2A, 2B completion status and changes made
  → Saves checkpoint-1
  
Orchestrator starts fresh session:
  /compact load checkpoint-1
  → Restores critical context
  
Step 3 continues with fresh context window
```

**Why:** Copilot's native checkpoint feature prevents loss on session restart

### Recommendation 3: Proactive Compaction Strategy

**What:** Don't wait for automatic compaction; trigger manually

**How:**
```javascript
// In Orchestrator
if (context_usage > 70%) {
  // Before starting next step
  agent.sendCommand('/compact focus on completed steps, skip old logs');
  
  // Then start new step
  agent.executeStep(step_3);
}
```

**Why:** You control when loss occurs, can ensure critical data is preserved

### Recommendation 4: Reduce Per-Agent Context Footprint

**Strategy:** Each agent gets MINIMAL context

**Current (Bloated):**
```
Agent 2B gets:
  - Full migration-state.json (3K)
  - Full breaking-changes.md (8K)
  - Full previous 2 steps log (5K)
  - Full project codebase scan (10K)
  → 26K tokens just for setup
```

**Optimized:**
```
Agent 2B gets:
  - Current step assignment (200 tokens)
  - Only relevant breaking changes for inject() (2K)
  - Only current files to modify (3K)
  - Path to migration-state.json (reference, not full content)
  → 5K tokens (80% reduction)
```

**Implementation:**
```javascript
// Instead of passing full state
const agent_context = {
  step_id: "2B",
  step_config: plan.steps.find(s => s.step_id === "2B"), // 200 tokens
  relevant_docs: reference_docs["dependency-injection"], // 2K
  files_to_modify: codebase.getInjectionFiles(), // 3K
  state_file_path: "./migration-state.json" // reference only
};

// Agent reads state from file, not context
```

### Recommendation 5: Streaming Large Outputs to Disk

**Leverage:** Copilot's feature to write large tool output to disk instead of context

**How:**
```javascript
// Test results → file, not context
const test_results = await agent.runTests(files);
fs.writeFileSync("./test-results-step-2B.json", test_results);

// In state file, reference:
execution_log.push({
  step: "2B",
  test_results_file: "./test-results-step-2B.json", // reference
  tests_passed: 30,
  tests_failed: 2
});

// Agent can read file if needed, doesn't consume context
```

### Recommendation 6: Hierarchical Compaction Strategy

**What:** Three-tier context management

**Tier 1: Active (Current Step)** ~30K
- Current step assignment
- Relevant reference docs
- Recent execution log

**Tier 2: Checkpoint (Last Phase)** ~10K
- Summary of completed steps
- Key decisions & findings

**Tier 3: Archive (Earlier Phases)** ~2K
- Link to checkpoint file (on disk)
- High-level summary only

**How:**
```
Start migration: Tier 1 = 30K
After Step 2B: Compact to Tier 2 (~10K), start fresh session with Tier 1
After Step 4: Compact to Tier 3 (~2K), reference checkpoint file
```

### Recommendation 7: Alternative: Use Claude Code or Cline Instead

**When You Should Consider This:**
- If migration frequently hits context limits
- If manual compaction becomes tedious
- If agent needs true multi-turn continuity

**Why:**
Claude Code and Cline use agent harnesses that can handle larger contexts; if you need 1M window, these are better than Copilot

**Trade-off:** Lose VS Code integration, but gain context capacity

---

## 6. Specific Recommendations for Your Angular Migration Skill

### For Orchestrator Agent:

**DO:**
- ✅ Keep `migration-state.json` on disk (reference, don't embed)
- ✅ Trigger `/compact` manually every 2-3 steps
- ✅ Pass only current step config to Orchestrator (not full plan)
- ✅ Stream test results to disk, reference in state file

**DON'T:**
- ❌ Pass full execution_log to each agent (only recent 5 entries)
- ❌ Include all reference docs (only current step's docs)
- ❌ Wait for automatic compaction (it's too late)
- ❌ Embed large tool outputs in context (use disk references)

### For Execution Sub-Agents:

**DO:**
- ✅ Receive minimal context (step config only)
- ✅ Read reference docs from `src/reference-docs/` (disk, not context)
- ✅ Read migration-state.json to determine scope (file I/O, not context)
- ✅ Write results to disk, update state file with references

**DON'T:**
- ❌ Include full codebase scan (query specific files instead)
- ❌ Pass full history of other steps
- ❌ Embed large diffs or test outputs in responses

### For Managing Across All Agents:

**Strategy: Rolling Summary**
```
Phase 1 (Steps 1-2B):   Tier 1 context, keep full details
                        ↓ /compact
Phase 2 (Steps 3-4):    Tier 2 checkpoint created, fresh context for new phase
                        ↓ /compact
Phase 3 (Verification): Tier 3 archive, reference checkpoint file only
```

---

## 7. Practical Implementation for Your Skill

### Configuration Settings

**Enable in VS Code:**
```json
// .vscode/settings.json
{
  "github.copilot.chat.summarizeAgentConversationHistory.enabled": true,
  "github.copilot.chat.compactionEnabled": true
}
```

**In Your Agent Code:**
```javascript
class Orchestrator {
  async run() {
    // Phase 1: Steps 1-2B
    await this.executePhase([step1, step2A, step2B]);
    
    // Proactive compaction before Phase 2
    if (this.contextUsage > 70%) {
      this.compactContext({
        focus: "completed steps 1, 2A, 2B",
        skip: "detailed logs from step 1",
        preserve: "dependency graph, file changes summary"
      });
    }
    
    // Phase 2: Steps 3-4 in fresh session
    await this.newSession("Phase 2: Verification");
    await this.executePhase([step3, step4]);
  }
}
```

### Monitoring Context Usage

**In Each Agent:**
```javascript
async executeStep(step) {
  const context_before = this.getContextUsage();
  
  // Execute step
  await this.doWork(step);
  
  const context_after = this.getContextUsage();
  const context_delta = context_after - context_before;
  
  // Log for analysis
  console.log(`Step ${step.id}: ${context_delta}K tokens used`);
  
  // Warn if approaching limit
  if (context_after > 80%) {
    console.warn(`Context at ${context_after}%: Consider compaction`);
  }
}
```

---

## 8. Comparison: Other Tools

### Claude Code (Native)
- **Context:** Can use full 1M window from Claude models
- **Best For:** Complex migrations with lots of context
- **Trade-off:** Not in VS Code IDE

### Cline (VS Code Extension)
- **Context:** Supports large contexts depending on backend
- **Best For:** VS Code + large context needs
- **Trade-off:** Need to install & configure separately

### Your Current Approach (Copilot Agents)
- **Context:** 128-192K with strategic management
- **Best For:** Quick interactive migrations (2-3 steps)
- **Trade-off:** Requires proactive compaction, state file management

---

## 9. Final Recommendation

**For Your Angular Migration Skill:**

**Tier 1 (Immediate - Recommended):**
Use Copilot agents with:
1. ✅ External state file (migration-state.json)
2. ✅ Proactive manual compaction every 2-3 steps
3. ✅ Minimal per-agent context (~5-10K tokens)
4. ✅ Disk-based output streaming (test results, logs)
5. ✅ Phase-based checkpointing

**Why:** Works well, fully integrates VS Code, minimal changes to your current design

**Cost:** ~30-40 min implementation time to add compaction strategy

---

**Tier 2 (If Tier 1 has issues):**
Switch to Claude Code:
1. Abandon VS Code integration
2. Use 1M context window natively
3. Simplify state management (less compaction needed)

**Why:** True infinite context for complex migrations

**Cost:** Lose VS Code IDE integration, ~4-6 hour rewrite

---

**Tier 3 (If you want both):**
Use Cline extension:
1. Runs in VS Code
2. Configurable backend (can use Claude Code)
3. Supports large contexts

**Why:** Best of both worlds

**Cost:** ~2-3 hour setup + configuration

---

## 10. Bottom Line

**Current State:** Copilot capped at 128K despite Claude supporting 1M; GitHub won't increase limits soon due to economics

**Your Best Move:** Combine Copilot agents with intelligent state file management + proactive compaction

**Timeline:** Add context management features this week, test with 19→20 migration by Monday

**Risk:** Low (your design already separates state from context)

**Effort:** ~2 days to implement compaction strategy + testing

---

## References

- GitHub community discussion on context window limits
- VS Code Copilot context management documentation (April 2026)
- VS Code v1.110 release notes (February 2026)
- GitHub Copilot CLI context management documentation

---

**Recommendation Status:** ✅ Ready to implement immediately

**Next Steps:** 
1. Add `/compact` triggers to Orchestrator (30 min)
2. Implement minimal-context pattern for sub-agents (2 hours)
3. Test with full 19→20 migration (2 hours)
4. Measure context usage & refine (1 hour)

**Total: 1 day of development**
