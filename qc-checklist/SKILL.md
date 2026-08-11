---
name: qc-checklist
description: Generates a human-focused QC checklist from an OpenSpec change for diff-based review of AI-generated implementations. Automatically triggers whenever the user asks to review code, inspect a PR or diff, evaluate implementation quality, or ask questions about code with the intent of code review or quality assurance. Identifies requirements, invariants, critical code areas, review depth, and targeted verification scenarios without approving or replacing human code review.
---

# `qc-checklist` Skill

The `qc-checklist` skill translates OpenSpec proposals and spec deltas into a structured, risk-focused **Human Quality Control (QC) Checklist**. It helps engineers conduct targeted, high-signal diff reviews of AI-generated implementations by identifying where critical code decisions live and where to focus line-by-line inspection.

---

## Automatic Invocation Trigger Criteria

This skill **must be automatically invoked** whenever the engineer expresses an intent to review code or inspect implementation quality.

> **Note on Manual Invocation:** Engineers can also manually trigger this skill at any time by calling it explicitly (e.g., `/qc-checklist`, _"Run the qc-checklist skill"_, or _"Generate a human QC checklist for this change"_).

### Trigger Examples (Intent-Based):

- _"Review this code / PR / diff."_
- _"Help me review the implementation for `<change-name>`."_
- _"Is this PR ready for human inspection?"_
- _"What should I pay attention to when reviewing this change?"_
- _"I need to check the quality of the AI-generated code for this spec."_
- Any prompt asking questions about changed code where the underlying goal is **verifying correctness, inspecting changes, or preparing for code review**.

---

## Non-Negotiable Guardrails

1. **NEVER approve PRs or certify code correctness.** This skill produces a review plan; the human engineer makes the quality judgment.
2. **NEVER modify application code or run auto-fix steps.**
3. **NEVER treat passing automated tests or AI self-reviews as a substitute for human inspection.**
4. **NEVER force equal review effort across all changed files.** Focus human attention on business logic, state changes, security, data integrity, and side effects.
5. **ALWAYS distinguish explicit specification requirements from inferred engineering risks.**

---

## Expected OpenSpec Directory Structure

OpenSpec organizes specs, changes, and AI context using a standardized file tree:

```text
openspec/
├── project.md                     # Global project context & tech stack
├── AGENTS.md                      # AI agent guidelines & workflow rules
├── specs/                         # Core source-of-truth specifications
│   └── <capability>/
│       └── spec.md                # Established capability contracts & behavior
└── changes/                       # Proposed modifications
    └── <change-name>/             # Target change folder for this review
        ├── proposal.md            # Problem context, business rationale, scope
        ├── design.md              # Technical design, architectural decisions, data models
        ├── tasks.md               # Work breakdown and task checklist
        └── specs/                 # Capability delta specifications
            └── <capability>/
                └── spec.md        # Requirement updates (ADDED/MODIFIED/REMOVED)
```

### Delta Spec Schema

Within `openspec/changes/<change-name>/specs/<capability>/spec.md`, OpenSpec documents changes using structured delta sections:

```markdown
## ADDED Requirements

### Requirement: FX Rate Required on Close

The user must supply a valid FX rate before initiating month closing.

#### Scenario: Valid FX rate provided

- GIVEN a valid FX rate of 1.35
- WHEN month close is executed
- THEN balance is revalued using 1.35

## MODIFIED Requirements

### Requirement: Account Balance Posting

- FROM: Posting goes to account 7100.
- TO: Posting goes to account 7110 for unrealized FX differences.

## REMOVED Requirements

- FROM: `### Requirement: Legacy Manual Rate Input`
- Reason: Superceded by automated rate lookup fallback.
```

---

## Operational Workflow

When invoked, execute the following procedure step by step:

```text
OpenSpec change folder (openspec/changes/<change-name>/)
      ↓
Read proposal.md (Intent & Scope)
      ↓
Read design.md (Architecture & Data Contracts)
      ↓
Read specs/<capability>/spec.md (Delta Requirements & Scenarios)
      ↓
Read tasks.md (Task Breakdown)
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

Locate and read all relevant OpenSpec artifacts inside `openspec/changes/<change-name>/`:

1. `proposal.md` – Scope, problem context, high-level intent.
2. `design.md` – Architectural decisions, schema migrations, side-effect guarantees.
3. `specs/<capability>/spec.md` – Delta requirements (`## ADDED`, `## MODIFIED`, `## REMOVED`) and test scenarios (`#### Scenario:`).
4. `tasks.md` – Planned tasks and subtasks.

If no specific change directory is supplied, inspect `openspec/changes/` to identify the active or target change folder.

---

### Step 2: Requirement & Invariant Extraction

Extract all functional requirements and explicit/implicit system invariants:

1. **Explicit Requirements:** Direct business rules, API contracts, state preconditions, and scenarios stated under delta specs.
   - _Example:_ "Retired cashflow fields should only appear when they have movement during the selected month."
2. **System Invariants:** Rules that must **always** remain true before, during, and after execution.
   - _Example:_ "Journal entries must balance debits and credits; historical closed-month entries must remain immutable."
3. **Inferred Engineering Risks:** Technical failure modes not explicitly named in the spec, but necessary for correctness.
   - _Example:_ "Ensure SQL joins cannot produce duplicate transaction rows and artificially inflate totals."

---

### Step 3: Diff-Oriented Targeting

Map every requirement and invariant to code location concepts where the requirement **must become executable code**:

- **Where does this logic live?** (e.g., service class, validation schema, SQL query, API controller, domain model).
- **Which side effects could occur?** (e.g., event publishing, DB mutations, third-party API calls, cache invalidation).

Ask the core heuristic question:

> _"If this code is wrong, could it materially violate a requirement, invariant, security property, data-integrity rule, or architectural constraint?"_

---

### Step 4: Review Depth Categorization

Classify implementation areas into three explicit review depths:

#### `DEEP` Review (Line-by-Line Inspection Required)

Apply to code that:

- Implements business or domain rules
- Enforces data invariants or transactional consistency
- Performs financial, mathematical, or rate-based calculations
- Handles authentication, authorization, or permission boundaries
- Mutates persistent data, performs deletes, or manages DB transactions
- Handles concurrency, locking, or race conditions
- Controls critical state transitions
- Performs critical input validation or security sanitation
- Triggers consequential external side effects

#### `NORMAL` Review (Flow & Decision Verification)

Apply to code that:

- Orchestrates high-level workflows (API -> Service -> DB)
- Transforms standard data models between layers
- Validates API contracts and request/response schemas
- Handles application error routing and messaging
- Manages standard application state propagation

#### `SKIM` Review (Mechanical / Structural Verification)

Apply to code that:

- Contains CSS or visual styling
- Directs simple UI component wiring or boilerplate rendering
- Imports modules or propagates static types
- Translates straightforward key-value mappings
- Consists of standard auto-generated boilerplate or config

---

### Step 5: Unexpected-Change & Risk Identification

Generate a targeted list of "red flag" items for the human reviewer to look out for during diff inspection:

- Unintended deletions or altered default parameters in existing routines.
- Omission of transactional wrappers (`BEGIN`/`COMMIT`) around multi-table mutations.
- Hidden performance pitfalls (e.g., $N+1$ queries, unindexed filter columns).
- Premature caching or missing cache invalidation.
- Silent error swallowing (empty `catch` blocks or default fallbacks masking failures).

---

### Step 6: Targeted Verification Scenario Construction

Draft concrete, reproducible manual test cases or verification steps that prove both happy-path correctness and edge-case resilience, anchoring them directly to the `#### Scenario:` sections in the spec deltas.

---

## Required Output Format

The skill MUST format its final output using the exact markdown structure below:

```markdown
# Human QC Checklist

## Change

- **OpenSpec change:** `<change-name>` (`openspec/changes/<change-name>/`)
- **Review scope:** `<short summary of change intent>`

## 1. Requirements to verify

### [ ] `<short requirement title>`

- **Requirement:** `<explicit requirement description from delta spec>`
- **Invariant:** `<invariant or N/A>`
- **Find in diff:** `<expected files/functions/modules>`
- **Review:** `DEEP | NORMAL | SKIM`
- **Verify:** `<specific condition to verify>`
- **Edge cases:** `<edge case scenario or N/A>`

_(Repeat for each extracted requirement)_

## 2. Critical implementation to read deeply

- **Location:** `<file / class / function / query>`
- **Why critical:** `<reason: e.g., financial math, transaction boundary, security risk>`
- **Questions to answer:**
  - `<question for human reviewer to answer when reading code>`
  - `<question for human reviewer to answer when reading code>`

_(Repeat for each critical code location)_

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
