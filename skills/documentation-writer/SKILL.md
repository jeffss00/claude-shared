---
name: documentation-writer
description: "Write and update project documentation for modules, APIs, and integrations. Use when asked to document a module, update docs, fill documentation gaps, or after completing a new feature. Triggers on: document, write docs, update docs, documentation gaps, doc this."
---

# Documentation Writer

Write and maintain project documentation that is question-driven, code-example-rich, and optimized for both human developers and AI coding assistants.

## Invocation Modes

This skill operates in two modes based on whether arguments are provided:

### Targeted Mode (with arguments)

User specifies what to document:

```
/documentation-writer auth module
/documentation-writer stripe integration
/documentation-writer /api/users endpoints
```

Workflow:
1. Read the source code for the specified target
2. Read existing doc if one exists (to update rather than overwrite)
3. Generate the doc using the appropriate template
4. Write or update the file in the correct `docs/` location

### Scan Mode (no arguments)

Auto-detect documentation gaps:

1. Read existing `docs/` directory structure (if any)
2. Walk the codebase to identify:
   - **Modules/packages** with no corresponding doc
   - **API route handlers** without endpoint documentation
   - **Third-party client usage** (SDK imports, API calls) without integration docs
   - **Stale docs** where the source has changed significantly since the doc was last written
3. Present findings to the user using `AskUserQuestion`:
   - List discovered gaps grouped by type (modules, APIs, integrations)
   - Let user select which items to document
   - Offer "Document all gaps" option
4. Generate docs for selected items

## Docs Directory Structure

All documentation lives under `docs/` at the project root:

```
docs/
  architecture.md          # System overview, module map, data flow
  design-system.md         # (frontend only) Design tokens, typography, color system
  modules/
    <module-name>.md       # Per-module documentation
    <heavy-module>/        # Subdirectory for modules with 3+ doc files
      overview.md
      <subtopic>.md
  apis/
    <endpoint-group>.md    # Internal API endpoint documentation
  integrations/
    <service-name>.md      # External API/service usage patterns
  guides/
    <topic>.md             # Developer conventions, patterns, how-tos
  archive/
    <old-file>.md          # Retired PRDs, plans, and outdated docs (reference only)
```

### When to use each directory

| Directory | Content | Example |
|-----------|---------|---------|
| `modules/` | What a module IS and how it works | "Creative2 manages ad creative optimization..." |
| `apis/` | Internal endpoint contracts and usage | "POST /api/assets/upload expects multipart..." |
| `integrations/` | How we use external services | "Our Stripe wrapper handles subscriptions via..." |
| `guides/` | Conventions and patterns to follow | "All routers use BrandController naming..." |
| `archive/` | Old plans and PRDs kept for reference | Superseded PRDs, completed plans |

**Module subdirectories:** When a module has 3+ documentation files, create a subdirectory instead of one large file. Use `overview.md` as the entry point.

Create directories as needed. Do not create empty placeholder files.

## Project Type Detection

Before writing docs, detect the project type to adapt terminology and patterns:

- **Check for** `package.json` with React/Next.js/Vue dependencies → frontend or fullstack
- **Check for** `requirements.txt`, `pyproject.toml`, `Pipfile` → Python backend
- **Check for** both frontend and backend markers → fullstack
- **Check for** `go.mod`, `Cargo.toml`, etc. → other backend

Adapt language accordingly:
- Frontend: "components", "hooks", "pages", "stores"
- Backend: "services", "handlers", "models", "middleware"
- Fullstack: use both vocabularies, note which layer each module belongs to

## Core Writing Principles

Every doc produced by this skill MUST follow these rules:

### 1. Question-Driven Structure

Organize content around "How do I..." questions that a developer working in this codebase would actually ask. Do NOT write encyclopedic API references. Write answers to real questions.

### 2. Code Examples Are Mandatory

Every "How do I..." section MUST include a working code example that:
- Can be copy-pasted into the codebase and work
- Includes necessary imports
- Shows the actual module/function being documented (not pseudocode)
- Uses real types, real function signatures, real file paths from the project

### 3. Brief Over Thorough

A short, accurate doc is better than a long, comprehensive one. Aim for docs that can be read in under 2 minutes. If a module is complex, focus on the 3-5 most common operations.

### 4. Update, Don't Overwrite

When a doc already exists:
- Read it first
- Preserve any manually-written notes, gotchas, or context
- Update code examples to match current source
- Add new sections for new functionality
- Remove sections for deleted functionality

## Document Templates

### Module Documentation (`docs/modules/<name>.md`)

```markdown
# <Module Name>

> One-line summary: what this module does and why it exists.

## Overview

2-3 sentences covering:
- What problem this module solves
- Key design decisions or patterns used
- Where it sits in the system (what calls it, what it calls)

## How do I...

### ...use this module?

[Working code example showing the most common usage pattern]

### ...configure it?

[Config options, environment variables, or setup required]

### ...<module-specific question>?

[Additional Q&A pairs based on actual code. Typically 2-5 of these.]

## Key Files

| File | Purpose |
|------|---------|
| `path/to/main.ts` | Brief description |
| `path/to/types.ts` | Brief description |

## Dependencies

**Uses:** List modules/services this depends on
**Used by:** List modules/services that depend on this
```

When generating "How do I..." questions for a module:
- Read all exported functions/classes
- Look at how the module is imported and used elsewhere in the codebase
- Focus on the operations that appear most frequently in consuming code
- Include error handling patterns if the module can fail

### API Endpoint Documentation (`docs/apis/<group>.md`)

```markdown
# <Endpoint Group> API

> One-line summary of what this group of endpoints handles.

## Endpoints

### `METHOD /path`

**Purpose:** What this endpoint does.

**Request:**
[Code example showing how to call this endpoint from the codebase]

**Response:**
[Example response shape with TypeScript type or Python dataclass]

**Notes:** Auth requirements, rate limits, or gotchas.

[Repeat for each endpoint in the group]

## How do I...

### ...call these endpoints from the frontend?

[Working code example using the project's actual HTTP client/fetch pattern]

### ...add a new endpoint to this group?

[Brief guide: which file, what pattern to follow, what middleware applies]

### ...handle errors from these endpoints?

[Error response shape and client-side handling pattern]
```

### Integration Documentation (`docs/integrations/<service>.md`)

```markdown
# <Service Name> Integration

> One-line summary: what we use this service for.

## Setup

Environment variables required:
| Variable | Description |
|----------|-------------|
| `SERVICE_API_KEY` | Brief description |

## How We Use It

### Our Client/Wrapper

[Code showing our wrapper, client initialization, or SDK setup]

### Common Patterns

[2-3 most frequent operations we perform with this service, with working code examples pulled from our codebase]

## Gotchas

- Specific issues, rate limits, quirks we've encountered
- Workarounds we've implemented

## Key Files

| File | Purpose |
|------|---------|
| `path/to/client.ts` | SDK wrapper / client setup |
| `path/to/service.ts` | Business logic using this service |
```

When documenting integrations:
- Search the codebase for imports of the service's SDK/package
- Find all files that use the client
- Identify the most common operations
- Note any retry logic, error handling, or rate limiting we've implemented
- Document environment variables by checking `.env.example`, config files, or deployment configs

### Architecture Overview (`docs/architecture.md`)

```markdown
# Architecture Overview

> One-line summary of the system.

## System Map

Brief description of major components and how they connect. If the project is fullstack, describe both layers.

## Modules

| Module | Location | Responsibility |
|--------|----------|---------------|
| Auth | `src/auth/` | Authentication and session management |
| Billing | `src/billing/` | Stripe integration, subscription logic |

## Data Flow

Describe the primary data flow(s) through the system. For web apps: request → middleware → handler → service → database. For pipelines: input → transform → output.

## External Services

| Service | Used For | Docs |
|---------|----------|------|
| Stripe | Payments | `docs/integrations/stripe.md` |
| OpenAI | Text generation | `docs/integrations/openai.md` |

## Key Decisions

Brief notes on important architectural choices and why they were made.
```

### Guide Documentation (`docs/guides/<topic>.md`)

```markdown
# <Convention/Pattern Name>

> One-line summary: what this guide covers and when to follow it.

## The Rule

Clear, direct statement of the convention or pattern.

## How to Follow It

### Pattern

[Working code example showing the correct pattern]

### Anti-Pattern

[Code example showing what NOT to do, with brief explanation of why]

## Examples in the Codebase

| File | How it follows this guide |
|------|--------------------------|
| `path/to/example.ts` | Brief description |

## Exceptions

When (if ever) it's acceptable to deviate from this guide.
```

When writing guides:
- Focus on ONE convention or pattern per guide
- Always include a correct and incorrect example
- Link to real files in the codebase that demonstrate the pattern
- Keep it short - guides are reference material, not tutorials

---

## Scan Mode: Gap Detection

When running in scan mode, use this process to find documentation gaps:

### 1. Discover Modules

- Glob for directories under `src/`, `app/`, `lib/`, `packages/`, or project-specific source roots
- Identify modules by looking for: index files, `__init__.py`, directories with 3+ related source files
- Compare against existing `docs/modules/` files

### 2. Discover API Endpoints

- Search for route handler patterns:
  - Next.js: `app/**/route.ts`, `pages/api/**`
  - Express: `router.get/post/put/delete`, `app.get/post`
  - FastAPI: `@app.get`, `@router.post`
  - Django: `urlpatterns`, `path()`
  - Flask: `@app.route`
- Compare against existing `docs/apis/` files

### 3. Discover Integrations

- Search `package.json` dependencies and `requirements.txt` for known service SDKs:
  - `stripe`, `@stripe/stripe-js`
  - `openai`, `anthropic`
  - `@aws-sdk/*`, `boto3`
  - `twilio`, `sendgrid`, `@sendgrid/mail`
  - `firebase`, `firebase-admin`
  - `redis`, `ioredis`
  - Other SDKs detected by import patterns
- Find files that import/use these packages
- Compare against existing `docs/integrations/` files

### 4. Discover Guides

- Check `ai-instructions/` or similar convention directories for undocumented patterns
- Look for repeated patterns in CLAUDE.md that could be extracted into standalone guides
- Check for linting configs, naming conventions, or architectural patterns without a corresponding guide
- Compare against existing `docs/guides/` files

### 5. Detect Stale Docs

- For each existing doc, find the source files it references
- Check if those source files have been modified more recently than the doc
- Flag docs where source has diverged significantly

## Post-Feature Documentation

When invoked after completing a feature (e.g., "document the feature I just built"):

1. Run `git diff --name-only HEAD~5..HEAD` (or appropriate range) to find recently changed files
2. Group changes by module/API/integration
3. For each group:
   - If doc exists: update it with new functionality
   - If no doc exists: create one
4. Update `docs/architecture.md` if a new module or integration was added

## Quality Checklist

Before saving any doc, verify:

- [ ] Every "How do I..." section has a working code example
- [ ] Code examples use actual imports, types, and file paths from the project
- [ ] No placeholder text like `[describe here]` or `TODO`
- [ ] Module dependencies are accurate (verified by checking imports)
- [ ] Environment variables listed actually exist in the project's config
- [ ] Doc is under 200 lines (split if longer)
- [ ] File is in the correct `docs/` subdirectory
- [ ] If updating, existing manually-written content is preserved
