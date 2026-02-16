**general-purpose, reusable audit prompt** you can use anytime another AI agent (or developer) implements a plan and you want Claude Code to rigorously review and correct it:

```markdown
# Audit & Correct Implementation

The implementation of an approved plan was completed by another AI agent.

Your role is NOT to redesign or reimplement from scratch.
Your role is to perform a deep technical audit and correct any issues.

---

## Scope

Review all files, changes, and related logic introduced or modified as part of this implementation.

Understand the original intent of the plan before making changes.

---

## Audit Objectives

1. Verify Architectural Compliance
   - Does the implementation strictly follow the approved design?
   - Is separation of concerns preserved?
   - Are responsibilities clearly isolated?

2. Detect Issues
   - Missing components
   - Partial or incomplete implementations
   - Incorrect assumptions
   - Broken abstractions
   - Tight coupling where it should not exist
   - Race conditions or concurrency risks
   - Non-idempotent behavior
   - Hidden side effects
   - Silent failure paths
   - Dead or unreachable code
   - Hardcoded values
   - Performance regressions

3. Validate Production Safety
   - Transaction safety
   - Error handling completeness
   - Retry and timeout logic
   - Graceful shutdown behavior (if applicable)
   - Backward compatibility
   - No breaking changes unless required

4. Validate Data & State Integrity
   - State transitions are correct
   - No unintended overwrites
   - Proper validation and guards
   - No inconsistent states possible

5. Validate External Integrations (if any)
   - Correct abstraction boundaries
   - No leaking implementation details
   - No direct infrastructure access from wrong layers

---

## Correction Rules

- Investigate before modifying.
- Make incremental, low-risk corrections.
- Preserve correct behavior.
- Treat deletions as high-risk and prove they are safe.
- Do not introduce speculative logic.
- Avoid hacks or temporary shortcuts.
- Improve clarity and maintainability where needed.

---

## Required Output

1. List all detected issues.
2. Explain why each issue is problematic.
3. Apply precise and minimal fixes.
4. Confirm final implementation aligns with the approved plan.
5. Confirm no regression risks remain.

Be strict and assume this is production-critical.
```

This version is:

* Architecture-agnostic
* Technology-agnostic
* Reusable across backend, frontend, infra, refactors, migrations
* Strong enough for production systems
* Structured enough to prevent shallow reviews

---

**general, reusable implementation prompt** you can give to GLM (or any AI agent) to implement a plan exported from Claude:

```markdown
# Implement Approved Plan

An architectural/technical plan has been approved.

Your task is to implement the plan exactly as written.

You are not redesigning it.
You are not simplifying it.
You are not replacing it with an alternative solution.

Follow the plan strictly.

---

## Before Writing Code

1. Read the entire plan carefully.
2. Identify:
   - Scope of changes
   - Files to create
   - Files to modify
   - Data model changes
   - Dependencies
3. Verify consistency with the existing codebase.
4. If anything in the plan conflicts with the current codebase, document it clearly before proceeding.

Do not assume.
Do not invent missing architecture.
Do not skip steps.

---

## Implementation Requirements

1. Follow the architecture exactly.
2. Maintain:
   - Correctness and stability
   - Clear separation of concerns
   - Clean, readable code
   - Incremental, safe changes
3. Preserve existing behavior unless explicitly instructed otherwise.
4. Treat deletions as high-risk:
   - Prove unused before removing.
5. Avoid:
   - Hardcoded values
   - Hacks or shortcuts
   - Silent failure handling
   - Speculative enhancements
6. Ensure:
   - Proper error handling
   - Idempotent operations where required
   - Transaction safety if applicable
   - Safe concurrency handling if applicable

---

## When Plan Is Ambiguous

If part of the plan is unclear:
1. Do not guess.
2. Identify the ambiguity explicitly.
3. Propose the safest interpretation.
4. Proceed conservatively.

---

## Code Quality Standards

- Follow existing project conventions.
- Do not introduce new patterns unless required by the plan.
- Keep abstractions clean.
- Avoid tight coupling.
- Minimize surface area of change.
- Ensure maintainability.

---

## Validation Before Completion

Before finalizing:

1. Re-read the plan.
2. Confirm every requirement is implemented.
3. Confirm nothing extra was added.
4. Confirm no regressions were introduced.
5. Ensure the system compiles/runs logically.

---

## Output

1. Summary of changes made.
2. Files created/modified.
3. Any risks or assumptions.
4. Confirmation that the implementation matches the approved plan.

Be precise. Assume this is production code.
```

Why this works well:

* Prevents “creative rewrites”
* Prevents speculative improvements
* Forces investigation first
* Reduces hallucinated architecture
* Keeps the implementing agent disciplined
* Creates a clear contract between planner and implementer


