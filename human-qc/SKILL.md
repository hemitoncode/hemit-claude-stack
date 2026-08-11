---
name: human-qc-review
description: Generates a human-focused QC checklist from an OpenSpec change for diff-based review of AI-generated implementations. Identifies requirements, invariants, critical implementation areas, review depth, and targeted verification scenarios without approving or replacing human code review.
---

# `human-qc-review` Skill

The `human-qc-review` skill translates OpenSpec proposals and specs into a structured, risk-focused **Human Quality Control (QC) Checklist**. It helps engineers conduct targeted, high-signal diff reviews of AI-generated implementations by identifying where critical code decisions live and where to focus line-by-line inspection.

---

## Non-Negotiable Guardrails

1. **NEVER approve PRs or certify code correctness.** This skill produces a review plan; the human engineer makes the quality judgment.
2. **NEVER modify application code or run auto-fix steps.**
3. **NEVER treat passing automated tests or AI self-reviews as a substitute for human inspection.**
4. **NEVER force equal review effort across all changed files.** Focus human attention on business logic, state changes, security, data integrity, and side effects.
5. **ALWAYS distinguish explicit specification requirements from inferred engineering risks.**

---

## Operational Workflow

When invoked, execute the following procedure step by step:

```
OpenSpec change
      ↓
Read proposal.md
      ↓
Read design.md
      ↓
Read specs/
      ↓
Read tasks.md
      ↓
Extract requirements & invariants
      ↓
Identify expected implementation areas
      ↓
Categorize review depth per component
      ↓
Generate human QC checklist
```

### Step 1: OpenSpec Artifact Discovery

Locate and read all relevant OpenSpec files associated with the specified change:
* `proposal.md` – Scope, problem context, high-level intent.
* `design.md` – Architectural decisions, data model changes, system flows.
* `specs/` (or spec updates/deltas) – Specific requirements, capabilities, and behaviors.
* `tasks.md` – Planned tasks, subtasks, and implementation steps.

If an OpenSpec directory is not explicitly supplied, locate the active OpenSpec context in the workspace (e.g., inside `openspec/`, `specs/`, or `.openspec/`).

---

### Step 2: Requirement & Invariant Extraction

Extract all functional requirements and explicit/implicit system invariants:

1. **Explicit Requirements:** Direct business rules, state requirements, UI/API behaviors stated in the spec.
   * *Example:* "Retired cashflow fields should only appear when they have movement during the selected month."
2. **System Invariants:** Conditions that must **always** remain true before, during, and after execution.
   * *Example:* "Account totals must equal the sum of line items; transaction entries must balance debits and credits."
3. **Inferred Engineering Risks:** Technical or systemic failure modes not explicitly named in the spec, but necessary for correctness.
   * *Example:* "Ensure SQL joins cannot produce duplicate rows and artificially inflate calculations."

---

### Step 3: Diff-Oriented Targeting

Map every requirement and invariant to code location concepts where the requirement **must become executable code**:

* **Where does this logic live?** (e.g., service class, validation schema, SQL query, API controller, domain model).
* **Which side effects could occur?** (e.g., event publishing, DB mutations, third-party API calls, cache invalidation).

Ask the core heuristic question:
> *"If this code is wrong, could it materially violate a requirement, invariant, security property, data-integrity rule, or architectural constraint?"*

---

### Step 4: Review Depth Categorization

Classify implementation areas into three explicit review depths:

#### `DEEP` Review (Line-by-Line Inspection Required)
Apply to code that:
* Implements business or domain rules
* Enforces data invariants or transactional consistency
* Performs financial, mathematical, or rate-based calculations
* Handles authentication, authorization, or permission boundaries
* Mutates persistent data, performs deletes, or manages DB transactions
* Handles concurrency, locking, or race conditions
* Controls critical state transitions
* Performs critical input validation or security sanitation
* Triggers consequential external side effects

#### `NORMAL` Review (Flow & Decision Verification)
Apply to code that:
* Orchestrates high-level workflows (API -> Service -> DB)
* Transforms standard data models between layers
* Validates API contracts and request/response schemas
* Handles application error routing and messaging
* Manages standard application state propagation

#### `SKIM` Review (Mechanical / Structural Verification)
Apply to code that:
* Contains CSS or visual styling
* Directs simple UI component wiring or boilerplate rendering
* Imports modules or propagates static types
* Translates straightforward key-value mappings
* Consists of standard auto-generated boilerplate or config

---

### Step 5: Unexpected-Change & Risk Identification

Generate a targeted list of "red flag" items for the human reviewer to look out for during diff inspection:
* Unintended deletions or altered default parameters in existing routines.
* Omission of transactional wrappers (`BEGIN`/`COMMIT`) around multi-table mutations.
* Hidden performance pitfalls (e.g., $N+1$ queries, unindexed filter columns).
* Premature caching or missing cache invalidation.
* Silent error swallowing (empty `catch` blocks or default fallbacks masking failures).

---

### Step 6: Targeted Verification Scenario Construction

Draft concrete, reproducible manual test cases or verification steps that prove both happy-path correctness and edge-case resilience.

---

## Required Output Format

The skill MUST format its final response using the exact structure below:

```markdown
# Human QC Checklist

## Change

- **OpenSpec change:** `<name_or_folder>`
- **Review scope:** `<short summary of change intent>`

## 1. Requirements to verify

### [ ] `<short requirement title>`

- **Requirement:** `<explicit requirement description>`
- **Invariant:** `<invariant or N/A>`
- **Find in diff:** `<expected files/functions/modules>`
- **Review:** `DEEP | NORMAL | SKIM`
- **Verify:** `<specific condition to verify>`
- **Edge cases:** `<edge case scenario or N/A>`

*(Repeat for each extracted requirement)*

## 2. Critical implementation to read deeply

- **Location:** `<file / class / function / query>`
- **Why critical:** `<reason: e.g., financial math, transaction boundary, security risk>`
- **Questions to answer:**
  - `<question for human reviewer to answer when reading code>`
  - `<question for human reviewer to answer when reading code>`

*(Repeat for each critical code location)*

## 3. Unexpected-change checks

- `<specific suspicious pattern or accidental regression to look for in the diff>`
- `<e.g., check that existing historical records are not mutated by new migrations>`

## 4. Targeted verification

[ ] `<concrete verification scenario or test case>`
[ ] `<concrete verification scenario or test case>`
[ ] `<concrete verification scenario or test case>`

## 5. Human review boundary

Deep-review the critical implementation areas above line-by-line. Skim mechanical code unless it appears suspicious. Use the requirements and invariants as the source of truth. Do not treat an AI review or passing tests as a substitute for human verification.
```

---

## Execution Guardrail Summary

> **Remember:** You are the guide, not the judge. Your goal is to optimize the engineer's cognitive load so they spend their time reviewing high-risk engineering decisions rather than skimming mechanical boilerplate.
