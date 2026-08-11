---
name: qc-followup
description: Assists the engineer during line-by-line code inspection by answering targeted technical questions about specific diff sections against the Human QC Checklist and OpenSpec invariants. Triggered explicitly by the engineer using /qc-followup <technical question about the code>.
---

# `qc-followup` Skill

The `qc-followup` skill acts as an interactive code-inspection assistant during human QC review. Once a `Human QC Checklist` has been generated via `/qc-checklist`, `qc-followup` helps the engineer evaluate specific code snippets, verify complex state transitions, and answer targeted checklist questions **without taking over the human review process or auto-approving code.**

---

## Invocation Commands

This skill is **explicitly invoked by the engineer** during code inspection:

- `/qc-followup <question or code snippet>`

### Example Usage:

- `"/qc-followup Look at lines 40-55 in src/services/month-close.ts. Does this catch block rollback the transaction if account 7110 fails?"`
- `"/qc-followup Help me answer question 2 under section 2 of the QC checklist for queries/ledger.sql"`

---

## Non-Negotiable Guardrails

1. **NEVER say "This code looks good" or "Approved."** Provide objective technical facts and execution paths; let the human render the judgment.
2. **DO NOT edit or modify application files.**
3. **DO NOT generate generic code quality feedback.** Focus strictly on the requirement, invariant, or risk being evaluated.
4. **ALWAYS anchor answers back to the OpenSpec specification and QC checklist.**

---

## Operational Workflow

When `/qc-followup` is invoked:

```text
Engineer asks question about specific code snippet or checklist item
      ↓
Locate active QC Checklist & OpenSpec invariants
      ↓
Read specified code file / diff snippet
      ↓
Analyze behavior against the specific invariant / requirement
      ↓
Report objective findings (Matches invariant / Diverges / Unclear edge case)
```

### Step 1: Context Grounding

Identify:

1. The specific **Checklist Item / Invariant** being investigated.
2. The specific **Code Location / Diff snippet** provided by the engineer.

### Step 2: Invariant & Behavior Analysis

Compare the execution logic in the code against the OpenSpec invariant:

- **Control Flow:** Are edge cases handled (nulls, empty collections, zero values)?
- **Transaction Bounds:** Are DB mutations atomic where required?
- **State Integrity:** Does any execution path leave an entity in an invalid state?
- **Side Effects:** Are side effects (events, logs, external calls) executed conditionally or unconditionally as intended?

### Step 3: Objective Technical Feedback

Present the analysis using clear, non-judgmental findings:

- What the code explicitly does line-by-line.
- How that behavior aligns or contrasts with the invariant.
- Any hidden assumptions or potential edge-case gaps for the engineer to test manually.

---

## Required Output Format

The skill MUST respond using this template:

```markdown
# QC Follow-Up Inspection

- **Target Item:** `<Checklist Invariant item or reference title>`
- **Code Inspected:** `<file_path:line_numbers or function_name>`

## Invariant Reference

> `<quote exact requirement or invariant from OpenSpec>`

## Technical Observation

- **Execution Flow:** `<brief description of what the code actually does>`
- **Invariant Alignment:** `<Matches Discrepancy Potential case edge invariant observed |>`

## Deep-Dives & Edge Cases

- **`<Aspect inspect to>`:** `<Detail 'Line 42 Exception, X but catches does e.g., line not on rollback the transaction' what>`
- **`<Edge case>`:** `<What X happens if input occurs>`

## Suggested Human Action

`<Specific before check checking item manual off or run scenario test this to>`
```
