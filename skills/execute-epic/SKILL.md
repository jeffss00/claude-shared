---
name: execute-epic
description: "Orchestrate parallel agent implementation of a beans epic. Use when executing a PRD, running an epic, or implementing domain beans. Triggers on: execute epic, run epic, implement prd, execute prd, run prd."
---

# Execute Epic

Orchestrate parallel agent teams to implement a beans epic. Spawns implementation, review, fix, and documentation agents — each with a single role and fresh context.

---

## Input

```
/execute-epic <epic-bean-id>
```

The epic bean ID is required. The epic must have domain beans as children (created by the `/prd` skill).

---

## Step 1: Read & Classify

Query the epic and its children:

```bash
beans show <epic-id>
beans children <epic-id>
```

Classify each child bean into one of:

| Type | Pattern | Example |
|------|---------|---------|
| **Domain (implementation)** | `BE-IMPL:`, `FE-IMPL:`, or similar | `BE-IMPL: Backend Priority Support` |
| **Review** | `REVIEW-*:` | `REVIEW-BE: Review Backend Implementation` |
| **Documentation** | `US-DOC:` | `US-DOC: Update documentation` |

If beans don't follow this naming, classify by content: beans with `## Files` and `### US-XXX` sections are domain beans; beans that reference `/code-review-linus` are reviews; beans referencing `/documentation-writer` are docs.

---

## Step 2: File Overlap Check (Safety Gate)

Before spawning any agents, verify no two domain beans share files.

1. Extract the `## Files` section from each domain bean
2. Build a map: `file → [domain beans that touch it]`
3. If any file appears in more than one domain bean: **STOP and report to user**

```
FILE OVERLAP DETECTED:

  src/schemas/task.py → BE-IMPL (BEAN-002), FE-IMPL (BEAN-003)

Cannot run these domains in parallel. Options:
  1. Merge overlapping domains into one bean
  2. Move the shared file to one domain and adjust the other
```

Do NOT proceed until overlaps are resolved. This is a hard gate.

---

## Step 3: Bean Splitting (Optional)

If a domain bean is too large (>8 stories OR >15 files), split it at runtime:

1. Identify natural split points (e.g., models vs. routes, components vs. pages)
2. Create new child beans under the epic with the split content
3. Verify no file overlaps in the new split
4. Proceed with the smaller beans

This is optional. Most well-structured PRDs from `/prd` won't need splitting.

---

## Step 4: Spawn Implementation Agents

Spawn one agent per domain bean. Domain beans without cross-dependencies run in parallel.

### Agent spawning rules:

- Use the `Task` tool with `subagent_type: "general-purpose"`
- Each agent runs in a worktree (`isolation: "worktree"`) if domain beans can run in parallel
- If all domain beans are sequential (each blocked by the previous), worktrees are not needed

### Agent instruction template:

```
You are implementing a domain bean for an epic. Your complete work order is in the bean.

1. Read your bean: `beans show <domain-bean-id>`
2. Read the epic for context: `beans show <epic-id>`
3. Work through each ### US-XXX section top-to-bottom
4. For each story:
   a. Read and understand the acceptance criteria
   b. Implement using TDD (write test → implement → pass)
   c. Check off each acceptance criterion as you complete it
   d. Run tests after each story to verify nothing is broken
5. After all stories complete:
   a. Run the full test suite
   b. Update the bean: `beans update <domain-bean-id> -s completed --body-append "## Done\n\n- Summary of what was built"`
   c. Commit your work with a descriptive message

Files you own (from the bean's ## Files section):
<paste file list here>

Do NOT modify files outside your ## Files list.
```

### Parallel execution:

```
Domain beans with no dependencies → spawn simultaneously
Domain beans with --blocked-by   → spawn after blocker completes
```

Example with backend and frontend domains:

```
BE-IMPL (no dependencies)  →  spawns immediately
FE-IMPL (blocked by BE-IMPL) →  spawns after BE-IMPL completes
```

---

## Step 5: Spawn Review Agents

**ALWAYS a separate agent from the implementation agent.** Never reuse an implementation agent for review.

After a domain bean's implementation completes, spawn a fresh review agent:

### Review agent instruction template:

```
You are a code reviewer. You have fresh context — you did NOT write this code.

1. Load the /code-review-linus skill
2. Read the review bean: `beans show <review-bean-id>`
3. Read every file listed in the bean's ## Files section
4. Review the code as a cohesive system using Linus's five-layer decomposition:
   - Data structures: Is the core data modeled correctly?
   - Special cases: Can if/else branches be eliminated?
   - Complexity: Can code be simplified?
   - Destructiveness: Does anything break existing functionality?
   - Practicality: Solving real problems or imaginary ones?
5. Record a taste score for each file (Good taste / Acceptable / Garbage)
6. If issues found:
   - Document each finding in the review bean with file, line, and description
   - Update the bean: `beans update <review-bean-id> --body-append "## Review Findings\n\n- ..."`
   - Output: <review-issues-found/>
7. If no issues:
   - Mark complete: `beans update <review-bean-id> -s completed`
   - Output: <review-passed/>

Your ONLY job is review. Do NOT fix any code. Do NOT implement anything.
```

### Why separate agents:

The review agent gets maximum context for its specific job — reading code with fresh eyes. It isn't weighed down by implementation decisions, which lets it catch issues the implementer is blind to.

---

## Step 6: Fix Agents

**ALWAYS a separate agent from the review agent.** Never reuse a review agent for fixes.

If a review finds issues (`<review-issues-found/>`), spawn a fresh fix agent:

### Fix agent instruction template:

```
You are a fix agent. Your job is to make targeted fixes based on review findings.

1. Read the review findings: `beans show <review-bean-id>`
2. Read the original domain bean for context: `beans show <domain-bean-id>`
3. For each finding:
   a. Read the file mentioned
   b. Make the minimal targeted fix
   c. Run tests to verify the fix doesn't break anything
4. After all fixes:
   a. Run the full test suite
   b. Commit with message: "fix: address review findings for <domain>"

Do NOT re-review the code. Do NOT refactor beyond what the findings require.
Your ONLY job is fixing the specific issues identified.
```

### Why separate agents:

Reviews consume significant context analyzing code. The fix agent needs fresh context to be surgical — reading just the findings and making targeted edits. This mirrors what worked best in practice: fresh agents with clean context produce better fixes.

---

## Step 7: Re-Review

After fixes, spawn a **NEW review agent** (fresh context again) to verify the fixes:

1. Same instructions as Step 5
2. The re-review agent checks both the original criteria AND that findings were addressed

### Max 2 fix/review cycles:

```
Implementation → Review → Fix → Re-Review → Fix → Re-Review → ESCALATE
```

If issues persist after 2 cycles, escalate to the user:

```
ESCALATION: Review cycle limit reached for <domain-bean-id>

After 2 fix/review cycles, these issues remain unresolved:
  1. [finding description]
  2. [finding description]

The implementation agent, two review agents, and two fix agents could not resolve these.
Please review manually and decide how to proceed.
```

---

## Step 8: Doc Agent

After ALL reviews pass across ALL domains, spawn a documentation agent:

### Doc agent instruction template:

```
You are a documentation writer.

1. Load the /documentation-writer skill
2. Read the documentation bean: `beans show <doc-bean-id>`
3. Read the epic for context: `beans show <epic-id>`
4. Follow the acceptance criteria in the doc bean
5. Mark complete when done: `beans update <doc-bean-id> -s completed`
```

The doc agent only runs after all review beans are marked completed.

---

## Step 9: Complete Epic

After all children (domain beans, review beans, doc bean) are completed:

```bash
beans update <epic-id> -s completed --body-append "## Epic Completed

### Summary
- [number] domain beans implemented
- [number] review cycles completed
- Documentation updated

### Domain Results
- BE-IMPL: [brief summary]
- FE-IMPL: [brief summary]
- Reviews: All passed
- Documentation: Updated"
```

---

## Error Handling

### Agent crashes or times out

- Check the bean status. If still `in-progress`, the agent didn't finish.
- Spawn a new agent with the same bean. The new agent reads the bean and picks up where the previous one left off (or starts over, which is fine for domain-sized work).

### Test failures

- If tests fail during implementation, the implementation agent handles it (it's part of their job).
- If tests fail during fixes, the fix agent handles it.
- If tests fail and the agent can't resolve them, it should report the failure in the bean and stop. The orchestrator escalates to the user.

### Circular dependencies

- Detected during Step 1 (read & classify). If `beans children` shows circular `--blocked-by`, report to user and stop.

### Beans already in-progress

- If a domain bean is already `in-progress` when the executor starts, ask the user:
  ```
  AskUserQuestion({
    questions: [{
      question: "BEAN-002 (BE-IMPL) is already in-progress. How should we proceed?",
      header: "Conflict",
      options: [
        { label: "Skip it", description: "Leave it as-is and only execute remaining beans" },
        { label: "Reset and re-run", description: "Reset to todo and re-execute from scratch" },
        { label: "Abort", description: "Stop execution entirely" },
      ],
      multiSelect: false
    }]
  })
  ```

---

## Status Tracking

After each agent completes (implementation, review, fix, or doc), display a progress summary:

```
EPIC PROGRESS: PRD: Task Priority System (BEAN-001)

  BE-IMPL (BEAN-002)     [COMPLETED]  3 stories implemented
  FE-IMPL (BEAN-003)     [IN PROGRESS] Agent working...
  REVIEW-BE (BEAN-004)   [COMPLETED]  Passed - all files good taste
  REVIEW-FE (BEAN-005)   [PENDING]    Waiting for FE-IMPL
  US-DOC (BEAN-006)      [PENDING]    Waiting for reviews

  Progress: 2/6 beans complete
```

Update this display after each agent event (start, complete, or fail).

---

## Concrete Example: Full Walkthrough

Given an epic "PRD: Task Priority System" with these children:

```
BEAN-001 (epic)
├── BEAN-002  BE-IMPL: Backend Priority Support        (no deps)
├── BEAN-003  FE-IMPL: Frontend Priority UI            (blocked by BEAN-002)
├── BEAN-004  REVIEW-BE: Review Backend Implementation (blocked by BEAN-002)
├── BEAN-005  REVIEW-FE: Review Frontend Implementation(blocked by BEAN-003)
└── BEAN-006  US-DOC: Update documentation             (blocked by BEAN-004, BEAN-005)
```

### Execution flow:

```
1. Read & classify all children
2. File overlap check: BE-IMPL owns backend files, FE-IMPL owns frontend files → no overlap ✓
3. Spawn implementation agent for BE-IMPL (no dependencies, runs immediately)
4. BE-IMPL completes →
   a. Spawn review agent for REVIEW-BE
   b. Spawn implementation agent for FE-IMPL (unblocked now)
5. REVIEW-BE completes (passed) → BEAN-004 marked completed
6. FE-IMPL completes → Spawn review agent for REVIEW-FE
7. REVIEW-FE completes with issues → Spawn fix agent
8. Fix agent completes → Spawn re-review agent (new REVIEW-FE agent)
9. Re-review passes → BEAN-005 marked completed
10. Both reviews passed → Spawn doc agent for US-DOC
11. US-DOC completes → Mark epic BEAN-001 completed
```

### Agent separation in this example:

```
Agent 1: BE-IMPL implementer    (spawned step 3, terminated step 4)
Agent 2: REVIEW-BE reviewer     (spawned step 4a, terminated step 5)
Agent 3: FE-IMPL implementer    (spawned step 4b, terminated step 6)
Agent 4: REVIEW-FE reviewer     (spawned step 6, terminated step 7 — found issues)
Agent 5: FE-IMPL fix agent      (spawned step 7, terminated step 8)
Agent 6: REVIEW-FE re-reviewer  (spawned step 8, terminated step 9 — passed)
Agent 7: US-DOC doc writer      (spawned step 10, terminated step 11)

Total: 7 agents, each with a single role and fresh context.
```

---

## Checklist

- [ ] Epic bean ID provided and valid
- [ ] All children classified (domain / review / doc)
- [ ] File overlap check passed — no shared files between domain beans
- [ ] Implementation agents spawned — one per domain bean
- [ ] Implementation agents use worktrees when running in parallel
- [ ] Review agents are ALWAYS separate from implementation agents (fresh context)
- [ ] Fix agents are ALWAYS separate from review agents (fresh context)
- [ ] Re-review agents are ALWAYS new agents (fresh context)
- [ ] Max 2 fix/review cycles before escalating to user
- [ ] Doc agent spawned only after ALL reviews pass
- [ ] Epic marked completed after all children completed
- [ ] Progress displayed after each agent event
