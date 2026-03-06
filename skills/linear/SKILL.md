---
name: linear
description: Create, scope, and manage Linear issues for the ProfitLabs workspace. Use when creating issues, scoping work, managing epics, or working with Linear task management.
---

# Linear Task Management Skill

This skill defines how to create, scope, and manage Linear issues for the ProfitLabs workspace.

## Workflow States

Issues progress through these statuses:

```
Backlog → Scoping → Scoped → In Progress → In Review → Done
```

- **Backlog**: Raw ideas, unrefined. Minimal detail required.
- **Scoping**: Actively being defined. Research and open questions being resolved.
- **Scoped**: Ready for development. All acceptance criteria defined, open questions resolved.
- **In Progress**: Work has started.
- **In Review**: PR submitted, awaiting review.
- **Done**: Merged and verified.

## Creating Issues

### Quick Capture (Backlog)

For new ideas, bugs, or feature requests — use minimal structure:

```markdown
## Outcome
_One sentence: what's different when this is done?_

## Notes
_Any context, links, or initial thoughts_
```

**Rules:**
- Status: `Backlog`
- Assign appropriate labels (see Labels section)
- Link to project if obvious, otherwise leave unassigned
- Don't over-specify — details come during Scoping

### Bug Reports (Backlog)

```markdown
## Outcome
_What should work that currently doesn't?_

## Reproduction
_Steps to reproduce, or "Unable to reproduce consistently"_

## Expected vs Actual
- Expected:
- Actual:

## Context
_Error messages, logs, affected accounts/users_
```

**Rules:**
- Always add `Bug` label
- Include error messages or log snippets if available
- Link to relevant code if known

## Scoping Issues

When moving an issue from Backlog → Scoping, expand it using the Definition of Ready template.

### For Small Tasks (bugs, minor features)

```markdown
## Outcome
_One sentence: what's different when this is done?_

## Acceptance criteria
- [ ]
- [ ]

## Prior art
_Existing patterns or files to reference_
-

## Touch points
_Files that will be modified_
-

## Verification
_Commands or steps to confirm success_
```

### For Larger Work (features, infrastructure)

Use the full template:

```markdown
## Outcome
_One sentence: what's different when this is done?_

## Acceptance criteria
- [ ]
- [ ]
- [ ]

## Non-goals / Boundaries
_What should NOT be done or changed?_
-
-

## Open questions
_Must be resolved before moving to Scoped_
- [ ]
- [ ]

## Prior art
_Existing patterns or files to reference_
-
-

## Touch points
_Specific files/services that will be modified_
-
-

## Dependencies
_Issues, APIs, env vars, or data that must exist first_
-

## Verification
_Exact commands to confirm success_
```bash

```

## Stop conditions
_Pause and ask if:_
- Scope: Changes exceed touch points listed
- Ambiguity: Multiple valid patterns, unclear which to follow
- Environment: Missing env vars, credentials, or test data
- Conflict: Prior art differs from expected

## Execution plan
_Each step becomes a sub-issue_

1. **Sub-issue title** — Brief description
2. **Sub-issue title** — Brief description
```

### Research Before Scoping

Before fully scoping an issue, research the codebase to answer:

1. **Prior art**: What existing code handles similar functionality?
2. **Touch points**: Which files will need changes?
3. **Patterns**: What conventions should be followed?
4. **Dependencies**: What must exist first?

Use commands like:
```bash
# Find similar patterns
grep -r "class.*Agent" chatpy/agents/
grep -r "def.*service" alertspy/services/

# Check model patterns
grep -r "class.*Model" alertspy/models/

# Find test patterns
ls tests/unit/ tests/integration/
```

Update the issue with findings before moving to Scoped.

## Epics and Sub-Issues

**When to create an epic:**
- Work spans 3+ distinct units (each could be its own PR)
- Multiple files/services with different concerns
- Estimated effort > 1 day

**Epic structure:**
- Parent issue contains: Outcome, full context, execution plan listing sub-issues
- Each sub-issue is self-contained with its own acceptance criteria and verification
- Sub-issues should be completable independently (in order, but separately deployable)

**Creating sub-issues:**
1. Create child issues linked to parent via `parentId`
2. Each sub-issue uses the same template appropriate to its size
3. Title format: `Epic name: Sub-task description` or just clear standalone title
4. Sub-issues inherit project from parent but get their own labels

## Labels

### Required Labels

Always apply at least one type label:

| Label | When to use |
|-------|-------------|
| `Bug` | Something broken that worked before |
| `Feature` | New functionality |
| `Improvement` | Enhancement to existing functionality |
| `enhancement` | Technical improvement (refactor, performance) |

### Category Labels

Apply all that are relevant:

| Label | When to use |
|-------|-------------|
| `category:frontend` | React/Next.js UI changes |
| `category:backend` | Django/FastAPI changes |
| `category:agent-tools` | Pydantic-ai agents or tools |
| `category:data-pipeline` | Prefect, dlt, data processing |
| `category:infrastructure` | DevOps, Docker, deployment |

### Priority Labels

Only set if explicitly requested or obvious:

| Label | Meaning |
|-------|---------|
| `priority:urgent` | Drop everything |
| `priority:high` | This week |
| `priority:medium` | This sprint |
| `priority:low` | Eventually |

## Naming Conventions

### Issue Titles

- Start with area if helpful: `Frontend: ...`, `Facebook Agent: ...`
- Be specific: "Fix timezone in Facebook agent" not "Fix bug"
- For bugs: Describe the symptom, not the fix
- For features: Describe the capability, not the implementation

**Good:**
- `Fix timezone conversion when passing dates to Facebook agent`
- `Add daily performance summary emails with LLM analysis`
- `Refactor Creative Labels page to show account overview grid`

**Bad:**
- `Fix bug`
- `Update agent`
- `New feature`

### Branch Names

Linear auto-generates branch names. Use them as-is:
`jeffsnyder/pro-123-issue-title-slug`

## Moving Issues to Scoped

An issue is ready to move from Scoping → Scoped when:

- [ ] Outcome is clear and testable
- [ ] Acceptance criteria are specific and checkable
- [ ] All open questions are resolved (checkboxes checked)
- [ ] Prior art identified (know which patterns to follow)
- [ ] Touch points listed (know which files will change)
- [ ] Verification steps defined (know how to prove it works)
- [ ] For epics: Sub-issues created

## Common Patterns Reference

### Agent files
- Agent definition: `chatpy/agents/{name}_agent.py`
- Agent tools: `chatpy/agents/tools/{name}_tools.py`
- Tests: `tests/integration/test_{name}_agent.py`

### Service files
- Services: `alertspy/services/{domain}/{name}_service.py`
- Tests: `tests/unit/test_{name}_service.py`

### Model files
- Models: `alertspy/models/{name}.py`
- Admin: `alertspy/admin.py`

### Frontend files
- Pages: `app/[lang]/{route}/page.tsx`
- Components: `app/[lang]/{route}/components/{Name}.tsx`

### Task files
- Celery tasks: `processpy/tasks/{name}.py`
