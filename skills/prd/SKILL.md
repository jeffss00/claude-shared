---
name: prd
description: "Generate a Product Requirements Document (PRD) for a new feature. Use when planning a feature, starting a new project, or when asked to create a PRD. Triggers on: create a prd, write prd for, plan this feature, requirements for, spec out."
---

# PRD Generator

Create detailed Product Requirements Documents that are clear, actionable, and suitable for autonomous AI implementation via domain-bean-driven parallel execution.

---

## The Job

1. Receive a feature description from the user
2. Ask at least 3 clarifying questions using the `AskUserQuestion` tool. Continue asking until you have a clear understanding of the feature and scope.
3. Conduct a design review conversation with the user — confirm architecture, schemas, data flows, and surface assumptions before generating stories
4. Generate domain beans with stories internally (do NOT create beans yet)
5. Present domain beans and stories to the user for review using `AskUserQuestion` — only create beans after approval
6. Create an epic bean with the full PRD as its body (`beans create -t epic --body`)
7. Create domain beans with stories as sections, using `--parent` linking to the epic
8. All PRD content lives inside beans — **do NOT create a separate PRD.md file**

### Clarifying Questions (Step 2)

Use the `AskUserQuestion` tool for all clarifying questions. Each question gets structured options with labels and descriptions. Users click to select instead of typing.

Example:

```
AskUserQuestion({
  questions: [{
    question: "What is the scope of this feature?",
    header: "Scope",
    options: [
      { label: "Full-stack", description: "Backend API + Frontend UI" },
      { label: "Backend only", description: "API endpoints, schemas, tests" },
      { label: "Frontend only", description: "UI components consuming existing API" },
    ],
    multiSelect: false
  }]
})
```

Use `multiSelect: true` where multiple answers make sense (e.g., "Which platforms should this support?"). Use `multiSelect: false` for mutually exclusive choices.

Ask at least 3 questions. Think about architecture, systems design, user flow, scope. Batch up to 4 related questions per `AskUserQuestion` call.

### Design Review Phase (Step 3)

After clarifying questions, but BEFORE generating stories, conduct a design review conversation with the user. Walk through and confirm:

- **Architecture:** What components/modules are involved? Which existing patterns to follow?
- **Schema/model changes:** New fields, new models, migrations needed? Confirm field types, defaults, constraints
- **Systems design:** How do components interact? Data flow? API contracts between front/back?
- **Design decisions not specified:** Surface any assumptions being made and get explicit confirmation
- **Dependencies on existing code:** What existing code is being extended? Any breaking changes?

Use `AskUserQuestion` for each decision point that has discrete options. Use plain text conversation for open-ended design discussion.

This phase continues until user confirms the design is solid. The design decisions become part of the epic bean's Technical Considerations section.

### Story Review with User (Step 5)

After generating the domain beans + stories internally, present them to the user for review BEFORE creating beans:

1. Walk through each domain bean and its story sections
2. Show the proposed `## Files` list for each domain
3. Use `AskUserQuestion` to confirm:

```
AskUserQuestion({
  questions: [{
    question: "Ready to create these domain beans?",
    header: "Confirm",
    options: [
      { label: "Approve", description: "Create all domain beans as presented" },
      { label: "Request changes", description: "I want to modify some stories or domains" },
      { label: "Discuss", description: "Let's talk through specific stories first" },
    ],
    multiSelect: false
  }]
})
```

4. Only create beans after user approval

---

## Domain Bean Sizing (THE NUMBER ONE RULE)

**Unit of work = domain bean, not individual story.**

Group stories by file ownership: stories that share files go in the same domain bean. Typical domains: Backend, Frontend, Shared/Infrastructure.

Each domain bean contains multiple stories as `### US-XXX` sections. Stories within a domain bean execute sequentially (since they share files), but domain beans without dependencies can execute in parallel.

### Right-sized domain beans:

- **Backend domain:** Schema, models, API endpoints, server actions — all sharing the same backend files
- **Frontend domain:** UI components, pages, state management — all sharing the same frontend files
- **Shared/Infra domain:** Config, types, utilities shared across domains

### Each story within a domain bean must still be completable in ONE context window (~10 min of AI work)

Stories stay fine-grained as sections within the domain bean body:

```markdown
### US-001: Add priority field to database [ ]

**Description:** As a developer, I need to store task priority.

**Acceptance Criteria:**
- [ ] Add priority column with default 'medium'
- [ ] Generate and run migration
- [ ] Tests pass

### US-002: Create priority API endpoints [ ]

**Description:** As a frontend, I need API endpoints for priority.

**Acceptance Criteria:**
- [ ] GET /tasks returns priority field
- [ ] PATCH /tasks/:id accepts priority changes
- [ ] Tests pass
```

### `## Files` Section (MANDATORY)

Every domain bean MUST include a `## Files` section listing all files it creates or modifies:

```markdown
## Files

**Creates:**
- `src/models/priority.py`
- `src/migrations/003_add_priority.py`

**Modifies:**
- `src/routes/tasks.py`
- `src/schemas/task.py`
```

**No two domain beans may share files.** If two domains would touch the same file, merge them or restructure.

### Too big (MUST split):

A domain bean with >8 stories or >15 files should be split into sub-domains.

| Too Big                              | Split Into                                                  |
| ------------------------------------ | ----------------------------------------------------------- |
| "Build entire backend" (12 stories)  | "BE-Models" (schema, models) + "BE-API" (routes, handlers)  |
| "Build entire frontend" (10 stories) | "FE-Components" (atoms, molecules) + "FE-Pages" (page-level)|

**Rule of thumb:** If you cannot describe the domain's scope in 2-3 sentences, it is too big.

---

## CRITICAL: TDD is WITHIN Stories, NOT Separate Stories

**DO NOT create separate stories for "stub", "write tests", and "implement".**

TDD (Test-Driven Development) is a **development technique used DURING implementation** of a single story, NOT a way to structure stories themselves.

### WRONG (over-engineering):

```
US-001: Create ContentItem dataclass stub
US-002: Write tests for ContentItem
US-003: Implement ContentItem
```

### CORRECT (feature-focused):

```
US-001: Create ContentItem dataclass with validation
  - [ ] Define dataclass with required fields
  - [ ] Add field validation
  - [ ] Write tests for validation logic
  - [ ] Tests pass
  - [ ] [TypeScript projects] Typecheck passes
```

Each story is a **complete feature unit**. The implementer uses TDD internally (write test -> implement -> pass) but this is execution detail, not PRD structure.

**Stories are organized by FEATURE/DEPENDENCY, not by development phase.**

---

## TDD Execution Rules (For Implementers)

When implementing stories with TDD, follow these rules strictly:

### Tests MUST Import Production Code

Tests must import from the **actual production module path**, not inline implementations.

**WRONG - inline function in test file:**

```python
# test_validator.py
def validate_email(email):  # WRONG: Don't define here!
    return "@" in email

def test_validate_email():
    assert validate_email("test@example.com") == True
```

**CORRECT - import from production module:**

```python
# test_validator.py
from src.validator import validate_email  # Import production code

def test_validate_email():
    assert validate_email("test@example.com") == True
```

### The RED Phase = Compile/Import Error

In proper TDD, the "failing test" in the RED phase can be a **compile or import error** because the production code doesn't exist yet.

**TDD Sequence:**

1. **RED:** Write test that imports `from src.validator import validate_email`
   - Test fails with `ImportError: cannot import name 'validate_email'`
   - This IS the failing test - compile errors count!
2. **GREEN:** Create `src/validator.py` with `validate_email()` function
   - Test now passes
3. **REFACTOR:** Clean up if needed

### Create Production File Stubs First (Optional)

To avoid import errors during test writing, you MAY create an empty stub:

```python
# src/validator.py (stub)
def validate_email(email: str) -> bool:
    raise NotImplementedError()
```

Then the test fails with `NotImplementedError` instead of `ImportError` - both are valid RED states.

### NEVER Do This:

- Define the function inside the test file
- Use mocks/fakes for the code you're about to write
- Copy-paste implementation into test to "make it pass"

The test file should have **zero business logic** - only assertions against production code.

---

## Story Ordering (Dependencies First)

Stories within a domain bean execute top-to-bottom. Domain beans execute based on `--blocked-by` dependencies.

**Correct order within a backend domain bean:**

1. Schema/database changes (migrations)
2. Models and validation
3. Server actions / backend logic
4. API endpoints

**Correct order across domain beans:**

1. Shared/Infrastructure domain (types, config)
2. Backend domain (schema, API)
3. Frontend domain (UI consuming the API)

**Wrong order:**

```
Frontend domain bean runs before Backend domain bean that provides its API
```

---

## Domain Reviews (Quality Gates)

**Add one review bean per domain bean.** Each review runs as a separate agent with fresh context.

The review bean MUST include the explicit file list copied from the domain bean it reviews, and MUST instruct the agent to load `/code-review-linus`.

### When to Add Domain Reviews

- **Simple features** (1-2 domain beans): 1 review per domain
- **Complex features** (3+ domain beans): 1 review per domain + 1 final integration review

### Domain Review Format

```markdown
### REVIEW-BE: Review Backend Implementation [ ]

**Description:** Review all backend implementation files using code review skills with fresh context.

## Files

(Copy the complete file list from the domain bean being reviewed)

**Creates:**
- `src/models/priority.py`
- `src/migrations/003_add_priority.py`

**Modifies:**
- `src/routes/tasks.py`
- `src/schemas/task.py`

**Acceptance Criteria:**

- [ ] Load `/code-review-linus` skill
- [ ] Review all files listed above as a cohesive system
- [ ] Apply Linus's five-layer decomposition:
  - Data structures: Is the core data modeled correctly? Unnecessary copying?
  - Special cases: Can any if/else branches be eliminated through better design?
  - Complexity: Can indentation depth be reduced? Can line count be halved?
  - Destructiveness: Does anything break existing functionality?
  - Practicality: Are we solving real problems or imaginary ones?
- [ ] Record taste score for each modified file (Good taste / Acceptable / Garbage)
- [ ] Flag any test files with inline production code (Check 1-3 from skill)
- [ ] If issues found:
  - Document findings in the review bean
  - Output: `<review-issues-found/>`
  - Do NOT mark this review task [x]
- [ ] If no issues:
  - Mark this review task [x]
  - Output: `<review-passed/>`
```

---

## Step 4: Acceptance Criteria (Must Be Verifiable)

Each criterion must be something an agent can CHECK, not something vague.

### Good criteria (verifiable):

- "Add `status` column to tasks table with default 'pending'"
- "Filter dropdown has options: All, Active, Completed"
- "Clicking delete shows confirmation dialog"
- "Tests pass"
- "Typecheck passes" (TypeScript projects)

### Bad criteria (vague):

- "Works correctly"
- "User can do X easily"
- "Good UX"
- "Handles edge cases"

### Always include as final criteria:

```
"Tests pass"
```

### For TypeScript projects, also include:

```
"Typecheck passes"
```

### For stories that change UI, also include:

```
"Verify changes work in browser"
```

---

## PRD Structure

Generate the PRD with these sections:

### 1. Introduction

Brief description of the feature and the problem it solves.

### 2. Goals

Specific, measurable objectives (bullet list).

### 3. Design Decisions

Key architecture, schema, and systems design decisions confirmed during the design review phase.

### 4. Domain Beans with User Stories

Each domain bean contains related stories grouped by file ownership:

```markdown
## Backend Domain

### Files

**Creates:** [list]
**Modifies:** [list]

### US-001: [Title] [ ]

**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Criterion
- [ ] Tests pass

### US-002: [Title] [ ]

...
```

### 5. Non-Goals

What this feature will NOT include. Critical for scope.

### 6. Technical Considerations

Known constraints, existing components to reuse, design decisions from the review phase.

---

## Example: Domain-Bean-Based PRD

### Step 1: Create the epic bean (this IS the PRD)

```bash
beans create "PRD: Task Priority System" -t epic -d "## Introduction

Add priority levels to tasks so users can focus on what matters most. Tasks can be marked as high, medium, or low priority, with visual indicators and filtering.

## Goals

- Allow assigning priority (high/medium/low) to any task
- Provide clear visual differentiation between priority levels
- Enable filtering by priority
- Default new tasks to medium priority

## Non-Goals

- No priority-based notifications or reminders
- No automatic priority assignment based on due date
- No priority inheritance for subtasks

## Design Decisions

- Priority stored as enum column with 3 values (not numeric scale)
- Reuse existing badge component with color variants
- Filter state managed via URL search params
- No new API endpoints; extend existing task CRUD

## Technical Considerations

- Reuse existing badge component with color variants
- Filter state managed via URL search params"
```

### Step 2: Create domain beans parented to the epic

```bash
# Capture the epic ID from step 1 (e.g., BEAN-001)

# Backend implementation domain
beans create "BE-IMPL: Backend Priority Support" -t task --parent BEAN-001 -d \
"## Files

**Creates:**
- src/migrations/003_add_priority.py

**Modifies:**
- src/models/task.py
- src/schemas/task.py
- src/routes/tasks.py

### US-001: Add priority field to database [ ]

**Description:** As a developer, I need to store task priority so it persists across sessions.

**Acceptance Criteria:**
- [ ] Add priority column: 'high' | 'medium' | 'low' (default 'medium')
- [ ] Generate and run migration successfully
- [ ] Tests pass

### US-002: Add priority to task schema and API [ ]

**Description:** As a frontend, I need the API to accept and return priority.

**Acceptance Criteria:**
- [ ] Task schema includes priority field with validation
- [ ] GET /tasks returns priority
- [ ] PATCH /tasks/:id accepts priority changes
- [ ] Tests pass

### US-003: Add priority filtering endpoint [ ]

**Description:** As a frontend, I need to filter tasks by priority.

**Acceptance Criteria:**
- [ ] GET /tasks accepts ?priority= query param
- [ ] Returns only matching tasks
- [ ] Tests pass"

# Frontend implementation domain
beans create "FE-IMPL: Frontend Priority UI" -t task --parent BEAN-001 --blocked-by BEAN-002 -d \
"## Files

**Creates:**
- src/components/PriorityBadge.tsx
- src/components/PrioritySelector.tsx

**Modifies:**
- src/components/TaskCard.tsx
- src/components/TaskEditModal.tsx
- src/components/TaskList.tsx

### US-004: Display priority badge on task cards [ ]

**Description:** As a user, I want to see task priority at a glance so I know what needs attention first.

**Acceptance Criteria:**
- [ ] Each task card shows colored priority badge (red=high, yellow=medium, gray=low)
- [ ] Priority visible without hovering or clicking
- [ ] Tests pass
- [ ] Verify changes work in browser

### US-005: Add priority selector to task edit [ ]

**Description:** As a user, I want to change a task's priority when editing it.

**Acceptance Criteria:**
- [ ] Priority dropdown in task edit modal
- [ ] Shows current priority as selected
- [ ] Saves immediately on selection change
- [ ] Tests pass
- [ ] Verify changes work in browser

### US-006: Filter tasks by priority [ ]

**Description:** As a user, I want to filter the task list to see only high-priority items when I'm focused.

**Acceptance Criteria:**
- [ ] Filter dropdown with options: All | High | Medium | Low
- [ ] Filter persists in URL params
- [ ] Empty state message when no tasks match filter
- [ ] Tests pass
- [ ] Verify changes work in browser"

# Backend review (blocked by BE-IMPL)
beans create "REVIEW-BE: Review Backend Priority Implementation" -t task --parent BEAN-001 --blocked-by BEAN-002 -d \
"## Files

**Creates:**
- src/migrations/003_add_priority.py

**Modifies:**
- src/models/task.py
- src/schemas/task.py
- src/routes/tasks.py

**Acceptance Criteria:**
- [ ] Load /code-review-linus skill
- [ ] Review all files listed above as a cohesive system
- [ ] Apply Linus's five-layer decomposition
- [ ] Record taste score for each file
- [ ] If issues: document and output <review-issues-found/>
- [ ] If clean: mark complete and output <review-passed/>"

# Frontend review (blocked by FE-IMPL)
beans create "REVIEW-FE: Review Frontend Priority Implementation" -t task --parent BEAN-001 --blocked-by BEAN-003 -d \
"## Files

**Creates:**
- src/components/PriorityBadge.tsx
- src/components/PrioritySelector.tsx

**Modifies:**
- src/components/TaskCard.tsx
- src/components/TaskEditModal.tsx
- src/components/TaskList.tsx

**Acceptance Criteria:**
- [ ] Load /code-review-linus skill
- [ ] Review all files listed above as a cohesive system
- [ ] Apply Linus's five-layer decomposition
- [ ] Record taste score for each file
- [ ] If issues: document and output <review-issues-found/>
- [ ] If clean: mark complete and output <review-passed/>"

# Documentation (blocked by BOTH reviews, not implementations)
beans create "US-DOC: Update documentation" -t task --parent BEAN-001 \
  --blocked-by BEAN-004 --blocked-by BEAN-005 -d \
"**Description:** Run the documentation-writer skill to update project docs for this feature.

**Acceptance Criteria:**
- [ ] Invoke /documentation-writer targeting all modules/APIs changed in this epic
- [ ] New modules have docs created in docs/modules/
- [ ] New API endpoints have docs created in docs/apis/
- [ ] docs/architecture.md updated if new module or integration was added
- [ ] All docs have working code examples with real imports
- [ ] Existing docs updated to reflect any changed interfaces"
```

### Step 3: View the PRD anytime

```bash
beans show BEAN-001          # View the full PRD
beans show BEAN-001 --raw    # View raw markdown
beans roadmap                # See the full feature roadmap
```

---

## Bean Creation Rules

1. **Create an epic bean** with the full PRD as its body (Introduction, Goals, Design Decisions, Non-Goals, Technical Considerations)
2. **Create one domain bean per domain** with all related stories as `### US-XXX` sections and a mandatory `## Files` section
3. **Create one review bean per domain bean** with the file list copied from the domain bean and instructions to load `/code-review-linus`
4. **Set dependencies** between beans (`--blocked-by`, `--blocking`): frontend blocked by backend, reviews blocked by implementations, docs blocked by reviews
5. **All beans start as `todo`** (not `in-progress`)
6. **Always create a documentation bean** as the final task, blocked by all review beans (see Step 6)

### Updating the PRD

All PRD updates happen through beans, never by editing a separate file:

```bash
# Replace the entire PRD body
beans update <epic-id> --body "new full content"

# Append to the PRD (e.g., adding a new section)
beans update <epic-id> --body-append "## New Section\n\nContent..."

# Find and replace within the PRD body
beans update <epic-id> --body-replace-old "old text" --body-replace-new "new text"

# Or edit the bean's markdown file directly in .beans/
# Find it with: beans show <epic-id> --raw
```

### Updating Domain Beans

When a domain's requirements change, update the domain bean:

```bash
# Update stories or file list
beans update <domain-id> --body "updated content..."

# Add implementation notes after completion
beans update <domain-id> --body-append "\n\n## Implementation Notes\n\n- What was built\n- Files modified"

# Mark complete with summary
beans update <domain-id> -s completed --body-append "\n\n## Done\n\n- Summary of changes"
```

### Bean Body Format

**Epic bean body** = the full PRD (Introduction, Goals, Design Decisions, Non-Goals, Technical Considerations).

**Domain bean body** = `## Files` section + `### US-XXX` story sections with acceptance criteria:

```
## Files

**Creates:**
- path/to/new_file.py

**Modifies:**
- path/to/existing_file.py

### US-001: [Title] [ ]

**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Specific verifiable criterion
- [ ] Tests pass
```

**Review bean body** = `## Files` section (copied from domain bean) + review acceptance criteria with `/code-review-linus` loading instruction.

This ensures subagents have full context when picking up a bean via `beans show <id>`.

---

## Step 6: Documentation Beans (Mandatory)

**Every epic MUST have a documentation bean as its final task.** The epic cannot be marked complete until documentation is done.

### Epic-Level Documentation Bean (Always Required)

Create this as the last bean in every epic, blocked by all review beans:

```bash
beans create "US-DOC: Update documentation" -t task --parent <epic-id> \
  --blocked-by <review-bean-1> --blocked-by <review-bean-2> -d \
"**Description:** Run the documentation-writer skill to update project docs for this feature.

**Acceptance Criteria:**
- [ ] Invoke /documentation-writer targeting all modules/APIs changed in this epic
- [ ] New modules have docs created in docs/modules/
- [ ] New API endpoints have docs created in docs/apis/
- [ ] New integrations have docs created in docs/integrations/
- [ ] docs/architecture.md updated if new module or integration was added
- [ ] All docs have working code examples with real imports
- [ ] Existing docs updated to reflect any changed interfaces"
```

### Story-Level Documentation (For Larger Stories)

For stories that introduce a **new module, new API endpoint group, or new integration**, add a documentation acceptance criterion directly to that story:

```
- [ ] Run /documentation-writer for [module/API/integration name]
```

This ensures docs are written while the context is fresh, not deferred to the end.

### When to Add Story-Level Doc Criteria

| Story introduces...                  | Add doc criterion? |
| ------------------------------------- | ------------------ |
| New module or package                 | Yes                |
| New API endpoint group                | Yes                |
| New external service integration      | Yes                |
| Changes to existing module internals  | No (covered by epic-level doc bean) |
| UI-only changes                       | No                 |
| Bug fix                               | No                 |

The epic-level doc bean is the final gate. Even if individual stories wrote docs, the epic doc bean verifies everything is consistent and complete across the whole feature.

---

## Output

1. Ask clarifying questions using `AskUserQuestion`
2. Conduct design review conversation with user
3. Generate domain beans + stories internally
4. Present to user for review, iterate until approved
5. Create epic bean with full PRD as its body
6. Create domain beans for all domains, parented to the epic
7. Create review beans for each domain, blocked by the corresponding domain bean
8. Create documentation bean as the final task, blocked by all review beans
9. Report bean IDs back to the user

---

## Checklist Before Saving

- [ ] Asked clarifying questions using `AskUserQuestion` tool (not text-based "1A, 2B" format)
- [ ] Conducted design review with user before story generation
- [ ] Incorporated user's answers and design decisions
- [ ] Stories reviewed with user before bean creation
- [ ] User stories use US-001 format within domain bean sections
- [ ] **Every task header has `[ ]` checkbox** (e.g., `### US-001: Title [ ]`)
- [ ] Each story completable in ONE iteration (small enough)
- [ ] Stories ordered by dependency within each domain bean
- [ ] Domain beans ordered by dependency across domains (shared -> backend -> frontend)
- [ ] All criteria are verifiable (not vague)
- [ ] Every story has "Tests pass" as criterion (and "Typecheck passes" for TypeScript projects)
- [ ] UI stories have "Verify changes work in browser"
- [ ] Non-goals section defines clear boundaries
- [ ] **No two domain beans share files**
- [ ] **Each domain bean has a `## Files` section**
- [ ] **Review beans have explicit file list copied from domain bean**
- [ ] **Review beans instruct agent to load `/code-review-linus`**
- [ ] **PRD written into epic bean body (no separate PRD.md)**
- [ ] **All domain beans parented to epic with `--parent`**
- [ ] **Dependencies set correctly (`--blocked-by`/`--blocking`)**
- [ ] **Documentation bean created as final task, blocked by all review beans**
- [ ] **Stories introducing new modules/APIs/integrations have doc criterion**
- [ ] **Stories are feature units, NOT TDD phases (no stub/test/implement split)**
- [ ] **TDD rules: tests import production code, no inline implementations**
