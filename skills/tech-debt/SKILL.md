---
name: tech-debt
description: Manage the running tech debt tracker. View open items, add new items from reviews, pick items to fix, and update status. Use when triaging tech debt, adding review findings, or working on debt reduction.
---

# Tech Debt Tracker Skill

Manage the running tech debt tracker beans across both projects. The tracker is an epic-level bean in each project that holds a structured list of tech debt items discovered during code reviews, PR feedback, and development.

## Bean Locations

| Project | Bean ID | File |
|---------|---------|------|
| **Backend** (alerts-py) | `alerts-py-uvnc` | `.beans/alerts-py-uvnc--tech-debt-tracker.md` |
| **Frontend** (alerts-frontend-dashtail) | `alerts-frontend-dashtail-aq72` | `.beans/alerts-frontend-dashtail-aq72--tech-debt-tracker.md` |

## Item ID Convention

- Backend items: `TD-BE-XXX` (e.g., `TD-BE-001`)
- Frontend items: `TD-FE-XXX` (e.g., `TD-FE-001`)

Increment the number from the last item in the respective bean.

## Workflows

### View open tech debt

Read both tracker beans and summarize open items:

```bash
# Read both trackers
cat /mnt/HC_Volume_104508259/code/alerts-py/.beans/alerts-py-uvnc--tech-debt-tracker.md
cat /mnt/HC_Volume_104508259/code/alerts-frontend-dashtail/.beans/alerts-frontend-dashtail-aq72--tech-debt-tracker.md
```

Present a summary table of open items with ID, severity, and one-line description.

### Add a new tech debt item

When a code review, PR, or development session surfaces tech debt, add it to the appropriate tracker bean under `## Open Items`.

Use this template:

```markdown
### TD-{BE|FE}-{NNN}: {Short title}
- **Source**: {Where it was found — e.g., "Code review on PR #123"}
- **Severity**: {High | Medium | Low}
- **Files**: {List of affected files with line numbers if relevant}
- **Impact**: {What happens if this isn't fixed}
- **Fix**: {Brief description of the recommended fix}
- **Child beans**: (none yet)
```

### Pick an item to fix

1. Read the tracker bean to pick an item
2. Create a child bean in the project directory for the specific fix:
   ```bash
   cd /mnt/HC_Volume_104508259/code/{project} && beans create "Fix: {item title}" -t task -d "Tech debt item {TD-XX-NNN}. {description}" -s in-progress
   ```
3. Do the work
4. Mark the child bean completed with a summary of changes
5. Update the tracker bean:
   - Add the child bean ID to the item's `Child beans` field
   - Move the item from `## Open Items` to `## Completed Items`
   - Add a completion note with the PR number

### Add findings from a code review

After running a code review (e.g., via `feature-dev:code-reviewer`), scan the output for items that are:
- Not blocking the current PR
- Worth fixing separately (duplication, missing tests, unsafe patterns, missing wiring)

Add each as a new item to the appropriate tracker bean. Skip trivial style nits.

## Severity Guide

| Level | Meaning | Examples |
|-------|---------|---------|
| **High** | Functional gap or bug risk | Missing field wiring, broken execution path |
| **Medium** | Maintenance burden or code quality | Duplication, missing tests for important paths |
| **Low** | Minor improvement | Unsafe casts, style inconsistencies, extra validation |

## Rules

- Always create child beans in the **project directory**, never the parent `/code` dir
- Cross-reference between backend and frontend trackers when items span both
- When completing an item, include the PR number and child bean ID in the completion note
- Don't let the tracker grow unbounded — periodically review and scrub items that are no longer relevant
