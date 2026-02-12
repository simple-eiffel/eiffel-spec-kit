---
name: eiffel-design-scan
description: Read-only design scanner. Finds abstraction, MI, and generics improvement opportunities in Eiffel libraries. Produces structured report. Use with /eiffel.design-scan command.
allowed-tools: Read, Grep, Glob, Bash, Task
---

# /eiffel.design-scan - Design Improvement Scanner

**Purpose:** Scan an Eiffel library to discover design improvement opportunities in three categories: missing abstractions, multiple inheritance candidates, and generics candidates. This is READ-ONLY — no source files are modified.

## Usage

```
/eiffel.design-scan <library-path>
```

**Example:**
```
/eiffel.design-scan d:\prod\simple_sorter
```

If no path provided, ask user: "Which library? Provide the full path (e.g., d:\prod\simple_sorter)"

## Project Scoping

```
<library-path>/
├── .eiffel-workflow/
│   └── evidence/
│       └── design-scan.md          ← OUTPUT (the only file written)
├── src/
│   └── *.e                         ← SCANNED (read-only)
├── test/
│   └── *.e                         ← SCANNED (read-only)
└── <library>.ecf                   ← SCANNED (read-only)
```

## Design Ethics (ISE EiffelBase Reference)

These principles from ISE EiffelBase guide every finding:

1. **Flat-and-Wide over Deep** — Prefer multiple role-based parents over deep single-parent chains (HASH_TABLE has 5 parents, not a 5-level chain)
2. **Deferred as Contract** — Abstract classes define WHAT via contracts, not HOW. Postconditions mandatory on deferred features.
3. **One Deferred Feature = One Responsibility** — Each deferred feature represents exactly one decision a descendant must make
4. **Generic Constraints Guide Implementation** — Constraints are design decisions. `[G -> COMPARABLE]` means "this class needs ordering"
5. **Feature Adaptation is Normal** — rename/redefine/undefine/select are standard MI tools, not code smells
6. **like Current Preserves Type** — For binary operations where both operands share the same type
7. **Invariants Enforce Laws** — Class invariants constrain all descendants uniformly
8. **Cursor Separates Concern** — Iteration is delegated to ITERATION_CURSOR, not built into structure

## Workflow

### Step 1: Inventory All Source Files

Read every `.e` file in `<library-path>/src/`. For each class, record:

- **Class name** and generic parameters
- **Inherit clause** — all parents, any feature adaptation (rename/redefine/undefine/select)
- **Feature signatures** — name, parameters, return type
- **Contracts** — require/ensure/invariant clauses
- **Deferred features** — which features are deferred vs effective
- **Feature categories** — group headings (Initialization, Access, Status report, etc.)

Build a mental model of the class hierarchy and feature landscape.

**Efficiency:** Use Grep to locate structural elements quickly before deep-reading:
```bash
# Find all class declarations
grep -rn "^class\|^deferred class\|^expanded class" <library-path>/src/

# Find all inherit clauses
grep -rn "^inherit" <library-path>/src/

# Find all generic parameters
grep -rn "class.*\[" <library-path>/src/

# Find deferred features
grep -rn "deferred$\|deferred " <library-path>/src/
```

### Step 2: Detect Abstraction Opportunities

Scan for these patterns:

**A1: Duplicate Feature Signatures**
- 2+ classes with features sharing the same name, parameter types, and return type
- Same contract (require/ensure) but different implementation → DEFERRED candidate
- Same contract AND same implementation → EFFECTIVE ancestor candidate

**A2: Deep Inheritance Chains**
- Chains >3 levels without deferred intermediaries
- Indicates missing abstraction layer

**A3: Template Method Candidates**
- One operation could generate others (e.g., defining `<` generates `<=`, `>`, `>=`)
- Multiple features that always co-occur and share a structural relationship
- Look for: comparison operators, encoding/decoding pairs, serialize/deserialize pairs

**A4: Missing Contracts on Shared Behavior**
- Features that exist in multiple classes without postconditions
- Opportunity: extract to deferred ancestor WITH contracts

For each finding, record:
```
Category: A1/A2/A3/A4
Classes: [list]
Features: [list with line numbers]
Description: [what the abstraction would capture]
Effort: [number of features to extract]
```

### Step 3: Detect MI Opportunities

Scan for these patterns:

**M1: Missing ITERABLE**
- Classes with internal collections (ARRAYED_LIST, HASH_TABLE, etc.) that don't inherit ITERABLE
- Should provide `new_cursor` for `across` loop compatibility

**M2: Duplicated Cross-Cutting Concerns**
- Same utility features (logging, validation, conversion) duplicated across unrelated classes
- Could be separate role-based parent

**M3: Missing Standard Parents**
- Classes that implement `<` but don't inherit COMPARABLE
- Classes that implement `hash_code` but don't inherit HASHABLE
- Classes with `out` but could benefit from CUSTOM_OUTPUT

**M4: Monolithic Classes**
- Large classes (>20 features) mixing distinct responsibilities
- Could be decomposed into role-based parents (storage vs traversal vs comparison vs serialization)

For each finding, record:
```
Category: M1/M2/M3/M4
Class: [target class]
Proposed Parent: [parent class name and role]
Features Affected: [list with line numbers]
Conflicts: [potential name clashes to resolve]
```

### Step 4: Detect Generics Opportunities

Scan for these patterns:

**G1: Concrete Types Where Generic Would Be Safer**
- Classes using `ANY` for element types
- Collections with hardcoded element types that could be parameterized
- Factory-like classes creating specific types

**G2: Missing Constraints on Existing Generics**
- `[G]` where `[G -> COMPARABLE]` needed because `<` is used on G
- `[G]` where `[G -> HASHABLE]` needed because `hash_code` is called on G
- Missing `detachable separate` for SCOOP compatibility

**G3: like Current Candidates**
- Binary operations where both operands and result should share the same type
- Builder-pattern features returning `like Current`

**G4: Type Parameter Extraction**
- A type mentioned in 5+ features that could become a class parameter
- Tight coupling to a specific type that limits reuse

For each finding, record:
```
Category: G1/G2/G3/G4
Class: [target class]
Current Type: [concrete type being used]
Proposed Parameter: [G/K/V with constraint]
Features Affected: [list with line numbers]
Client Impact: [files that would need updating]
```

### Step 5: Produce Structured Report

Create directory if needed:
```bash
mkdir -p <library-path>/.eiffel-workflow/evidence
```

Write `<library-path>/.eiffel-workflow/evidence/design-scan.md`:

```markdown
# Design Scan: <library-name>
# Date: [timestamp]
# Classes scanned: [count]
# Source files: [count]

## Summary

| Category | Findings | High | Medium | Low |
|----------|----------|------|--------|-----|
| Abstraction | [n] | [n] | [n] | [n] |
| MI | [n] | [n] | [n] | [n] |
| Generics | [n] | [n] | [n] | [n] |
| **Total** | **[n]** | **[n]** | **[n]** | **[n]** |

## Findings

| # | Priority | Category | Classes | Opportunity | Effort | Suggested Skill |
|---|----------|----------|---------|-------------|--------|-----------------|
| 1 | HIGH | abstract | A, B | Extract deferred X | 3 features | /eiffel.abstract |
| 2 | MEDIUM | mi | C | Add ITERABLE parent | 1 feature | /eiffel.mi |
| 3 | LOW | generics | D | Parameterize on G | 5 features | /eiffel.generify |

## Detailed Findings

### Finding 1: [title]
**Priority:** HIGH
**Category:** abstract
**Classes:** [class names with file paths]
**Description:** [what the opportunity is and why it matters]
**Evidence:**
- [class]:line [n]: [feature signature]
- [class]:line [n]: [feature signature]
**Suggested Action:** /eiffel.abstract targeting [classes]
**Effort:** [n] features to extract

[... repeat for each finding ...]

## Recommended Order

1. [Finding #] — [why first: highest impact / unblocks others]
2. [Finding #] — [why second]
3. ...
```

### Prioritization Rules

- **HIGH** = Duplicate code across classes / missing contracts / type safety gaps
- **MEDIUM** = Structural improvement / missing standard parents / convenience
- **LOW** = Nice-to-have / minor cleanup / cosmetic

## Completion

```
Design scan COMPLETE for: <library-path>
Report: <library-path>/.eiffel-workflow/evidence/design-scan.md

Findings: [n] total ([n] HIGH, [n] MEDIUM, [n] LOW)
  - Abstraction opportunities: [n]
  - MI opportunities: [n]
  - Generics opportunities: [n]
```

**Do NOT blindly list next steps.** The report speaks for itself — the user can read it and decide what to act on. Only mention a specific skill if the user asks "what now?"

## Context Management (RLM Pattern)

This skill focuses ONLY on: `<library-path>`

**DO NOT:**
- Read files outside this library directory
- Modify any source files (READ-ONLY skill)
- Load entire ecosystem into context

**DO:**
- Use Grep for structural element discovery before deep reading
- Use Task tool with Explore agent for EiffelBase pattern reference questions
- Focus on one category at a time to avoid context bloat

## Anti-Drift

- **READ-ONLY:** Do not modify any source files under any circumstances
- **Evidence Required:** Every finding must cite specific classes, features, and line numbers
- **No Imagination:** Do not invent findings — every opportunity must be grounded in actual code patterns
- **Prioritize by Impact:** HIGH = real duplication/safety issues, not subjective aesthetics
- **Skill Version Lock:** If you discover skill improvements during workflow, queue them in `<library-path>/.eiffel-workflow/skill-improvements.md` — do NOT modify skills mid-workflow
