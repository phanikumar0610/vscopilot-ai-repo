# Angular Migration Skill v2 - Requirements Prompt

**Use this prompt to brief another AI agent or team member on the complete project.**

---

## PROMPT: Angular Migration Skill v2 - Full Requirements

You are tasked with building a **multi-agent Angular migration framework** for automating version upgrades (19→20, 20→21, etc.) across monorepos, seed client apps, and mobile applications in a financial technology environment (air-gapped, secure, VS Code GitHub Copilot compatible).

### **Core Objective**
Reduce Angular migration time from **8-12 hours to 2-3 hours** (60-70% reduction) by automating version upgrades with intelligent parallelization, retry logic, and comprehensive technical guide generation.

### **Architecture: 7-Agent Design**

Build exactly 7 agents, each with specific responsibility:

1. **Pre-flight Agent**
   - Validates environment readiness (Node version, peer dependencies, Angular version compatibility)
   - Fetches fresh reference docs for target version
   - Identifies blockers
   - Output: Pre-flight report (ready/not-ready status)
   - User checkpoint: User confirms "Proceed? YES/NO"

2. **Migration Plan Agent**
   - Reads reference docs + scans current project codebase
   - **Builds dependency graph (DAG) UPFRONT** to determine which steps are sequential vs. parallel
   - Groups migration steps by change type:
     - Group A: Control Flow (template syntax: *ngIf → @if, *ngFor → @for)
     - Group B: Dependency Injection (inject() patterns)
     - Group C: Testing (Jasmine → Vitest)
     - Group D: Build & Config
   - Output: Structured migration plan (Markdown + JSON) with dependency graph
   - User checkpoint: User reviews plan and confirms "Accept? YES/REQUEST_CHANGES"

3. **Installation Agent**
   - Validates required dependencies are installed
   - Does NOT auto-install (user must install manually)
   - **3-Attempt Validation Loop:**
     - Attempt 1/3: Missing dep? Show user install command, user confirms completion
     - Attempt 2/3: Still invalid? Retry prompt
     - Attempt 3/3: Still invalid? Escalate to manual review
   - Log validation results

4. **Execution Orchestrator (Main Agent)**
   - Spawns sub-agents for parallel execution
   - **Uses file-based state coordination** (VS Code GitHub Copilot compatible)
   - Single source of truth: `migration-state.json` (atomic read/write, no race conditions)
   - **Polling loop** (every 10 seconds) to detect:
     - Step completion
     - Step failure
     - Timeout (>30 min per step)
   - **Retry logic:** MAX_RETRIES = 3
     - If sub-agent fails: increment retry_count, spawn new instance
     - If retry_count >= 3: escalate to Fix Agent
   - **Sequential vs. Parallel:** Respects dependencies from Migration Plan Agent
     - Step 1: Sequential (update dependencies)
     - Steps 2A + 2B: Run in parallel (independent changes)
     - Step 3: Sequential (depends on 2A + 2B completion)
   - Consolidates all execution logs into master migration plan document

5. **Verification Agent**
   - During migration: Run ONLY tests for modified modules (targeted, not full suite)
   - After all steps: Run full test suite + build verification
   - Log all test results

6. **Fix Agent**
   - Receives issue reports (from user or Verification Agent)
   - Analyzes issue, proposes fix with explanation
   - Applies fix to code
   - Runs ONLY targeted tests for affected module
   - User confirms: "Does this fix work? YES/NO"
   - If YES: logs fix, migration continues
   - If NO: escalates to manual review
   - Updates technical guide with fix details

7. **Summary Agent**
   - Generates technical change guide (final output)
   - Documents all changes made (by file, by step, by change type)
   - Documents all fixes applied post-migration
   - Extracts patterns & lessons learned for future migrations
   - Output: Markdown (human-readable) + JSON (structured data)

### **State Management Requirements**

- **Single source of truth:** `migration-state.json` file
- **Schema includes:**
  - migration_id, status, created_at, target_version
  - step_assignments (with depends_on, parallel_with)
  - steps_completed, steps_in_progress, steps_failed, steps_escalated
  - execution_log (appended by all agents with: step, status, files_modified, changes, tests_run, retry_count, duration_seconds)
- **Atomic operations:** Write to .tmp file, then rename (prevents corruption on crash)
- **Polling-based coordination:** Orchestrator polls every 10 seconds, not language-level shared state

### **Version-Agnostic Template**

- Skill code contains NO hardcoded version logic
- All breaking changes, syntax examples, migration steps come from **reference docs** (separate files)
- For Angular 19→20: read from `src/reference-docs/19-to-20/`
- For Angular 20→21: read from `src/reference-docs/20-to-21/`
- Same skill code works for all versions—only reference docs change

### **Reference Docs Structure**

For each version (e.g., 19→20), create folder: `src/reference-docs/19-to-20/`

Required files:
- `migration-document.md` — Master guide
- `breaking-changes.md` — All breaking changes table
- `control-flow.md` — Before/after syntax examples (*ngIf → @if, *ngFor → @for, *ngSwitch → @switch)
- `dependency-injection.md` — inject() patterns, provider scope
- `testing-migration.md` — Jasmine → Vitest migration
- `performance-notes.md` — Performance impact, deprecation timeline

Fetching: Fresh per migration (not cached)

Optional: Separate reference codebase repo with before/after code examples for pattern matching

### **Repository Structure**

```
src/
├─ agents/
│  ├─ orchestrator.js
│  ├─ preflight-agent.js
│  ├─ migration-plan-agent.js
│  ├─ installation-agent.js
│  ├─ execution-agent.js
│  ├─ verification-agent.js
│  ├─ fix-agent.js
│  └─ summary-agent.js
├─ state/
│  ├─ migration-state.json
│  ├─ state-manager.js
│  └─ types.ts
├─ utils/
│  ├─ retry-logic.js
│  ├─ dependency-detector.js
│  ├─ logger.js
│  └─ validators.js
└─ reference-docs/
   ├─ 19-to-20/ [all files above]
   └─ 20-to-21/ [all files above]
```

### **User Interaction Points**

1. **Pre-flight Confirmation** → User confirms "Proceed? YES/NO"
2. **Plan Review** → User confirms "Accept? YES/REQUEST_CHANGES"
3. **Installation** → User installs missing dependencies (max 3 prompts)
4. **Execution Monitoring** → Background polling (user sees progress)
5. **Issue Reporting** → User reports test failures for Fix Agent
6. **Fix Validation** → User confirms "Does this fix work? YES/NO"
7. **Final Report** → Technical guide generated

### **Retry & Failure Handling**

- **3-Attempt Retry Logic:**
  - Sub-agent fails → Orchestrator increments retry_count
  - retry_count < 3 → Spawn new sub-agent instance (same step)
  - retry_count >= 3 → Escalate to Fix Agent
- **Backoff Strategy:** Exponential (1s, 2s, 4s)
- **Step Timeout:** 30 minutes per step max
- **Escalation:** Fix Agent investigates, proposes fix, runs targeted tests

### **Performance Requirements**

| Metric | Target | Current | Improvement |
|--------|--------|---------|---|
| Total migration time | 2-3 hours | 8-12 hours | 60-70% reduction |
| Installation time | Batched (1 command) | Per-step installs | 90% reduction |
| Testing overhead | Targeted only | Full suite per-step | 80% reduction |
| Parallel efficiency | 2A + 2B concurrent | Sequential | 2x speedup for step 2 |

### **Scalability & Environment**

- **Monorepo support:** 3 monorepos (polyrepo, healthcare, WI/Enterprise)
- **Version support:** 19→20, 20→21, 21→22, etc.
- **App types:** Seed client, monorepo, mobile app
- **Environment:** Air-gapped, VS Code GitHub Copilot, local execution only
- **State file:** JSON file (no database required)
- **Logging:** Complete execution_log in migration-state.json

### **Acceptance Criteria**

✅ All 7 agents implemented & functional
✅ StateManager working with atomic operations
✅ Full 19→20 migration succeeds start-to-finish
✅ Parallel execution verified (2A + 2B concurrent)
✅ Retry scenario tested (step fails, retries, succeeds)
✅ Escalation tested (step fails 3x, Fix Agent invoked)
✅ Technical guide generated correctly
✅ Reference docs complete for 19→20
✅ Tests passing (unit + integration + end-to-end)
✅ CI/CD pipeline working
✅ Documentation complete
✅ Demo-ready (works on test project)

### **Implementation Phases**

**Phase 1 (1-2 days):** Setup
- Folder structure
- StateManager (critical foundation)
- RetryLogic

**Phase 2 (3-5 days):** Agents
- All 7 agents implemented with tests

**Phase 3 (2-3 days):** Integration
- End-to-end testing
- Parallel execution verified
- Retry scenarios tested

**Phase 4 (1-2 days):** Hardening
- Edge cases, performance testing
- Documentation, CI/CD

**Total: 2-4 weeks**

### **Success Metrics**

- ✅ Migration time reduced from 8-12h to 2-3h (60-70% reduction)
- ✅ Parallel execution provides 2x speedup for step 2
- ✅ 3-retry logic handles 95% of transient failures
- ✅ Technical guide 100% accurate (all changes documented)
- ✅ Skill scales to 3 monorepos (polyrepo, healthcare, WI/Enterprise)
- ✅ Version-agnostic (only reference docs change for new versions)

### **Documentation & Handoff**

Comprehensive documentation provided:
- `SKILL_SPEC_V2_UPDATED.md` — Complete architecture blueprint (46 KB)
- `COMPLETE_REQUIREMENTS.md` — Full requirements for implementation (33 KB)
- `SKILL_INTEGRATION_GUIDE.md` — Step-by-step implementation guide (30 KB)
- `QUICK_START.md` — 5-step quick reference (8.3 KB)
- Reference docs templates for 19→20 migration

### **Your Role**

1. Review all reference documentation (provided)
2. Implement the 7 agents following the specification
3. Create folder structure as defined
4. Write tests for each agent (unit + integration)
5. Verify acceptance criteria are met
6. Generate sample execution (19→20 migration on test project)
7. Deliver:
   - GitHub repo with all code
   - migration-state.json from sample run
   - technical-change-guide-19-to-20.md output
   - passing test suite

### **Key Decisions**

- ✅ File-based state coordination (VS Code GitHub Copilot compatible, no language-level shared state needed)
- ✅ 3-attempt retry logic with exponential backoff
- ✅ 7 separate agents (not monolithic)
- ✅ Version-agnostic skill (only reference docs version-specific)
- ✅ Atomic state file operations (no corruption on crash)
- ✅ Targeted testing (not full suite per-step, reduces time)
- ✅ Parallel steps (2A + 2B concurrent for efficiency)
- ✅ User controls all installations (security-first approach)

### **Questions to Clarify**

- Reference docs storage location (GitHub repo? CDN?)
- Target launch date for patch version support
- Preferred test framework (Jest? Mocha?)
- Exact breaking changes for 19→20 (for reference docs)
- Any specific VS Code GitHub Copilot limitations?

---

**Start with Phase 1 (StateManager + RetryLogic), then build Orchestrator, then individual agents. Reference COMPLETE_REQUIREMENTS.md for detailed acceptance criteria and exit conditions.**

**Timeline: 2-4 weeks for full implementation.**

**Success: Implement all 7 agents, verify parallel execution works, pass all acceptance criteria, deliver working 19→20 migration on test project.**
