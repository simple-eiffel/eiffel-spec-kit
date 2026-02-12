---
name: eiffel-mi
description: Apply Eiffel multiple inheritance with role-based parents. Uses feature adaptation (rename/redefine/undefine/select) to resolve conflicts. Use with /eiffel.mi command.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task
---

# /eiffel.mi - Multiple Inheritance Application

**Purpose:** Apply Eiffel multiple inheritance to add role-based parents, using feature adaptation (rename/redefine/undefine/select) to resolve conflicts. Every parent has a documented ROLE — no role means don't add it.

## Usage

```
/eiffel.mi <library-path>
```

**Example:**
```
/eiffel.mi d:\prod\simple_datetime
```

If no path provided, ask user: "Which library? Provide the full path (e.g., d:\prod\simple_datetime)"

## Project Scoping

```
<library-path>/
├── .eiffel-workflow/
│   └── evidence/
│       ├── design-scan.md        ← INPUT (optional, from /eiffel.design-scan)
│       └── mi-application.txt    ← OUTPUT (evidence)
├── src/
│   └── *.e                       ← MODIFIED (update inherit clauses, add role classes)
├── test/
│   └── *.e                       ← VERIFIED (existing tests must pass)
└── <library>.ecf                 ← MAY UPDATE (if new dependency added)
```

## Design Ethics (ISE EiffelBase Reference)

1. **Flat-and-Wide over Deep** — Prefer multiple role-based parents over deep single-parent chains. HASH_TABLE has 5 parents (TABLE, READABLE_INDEXABLE, TABLE_ITERABLE, MISMATCH_CORRECTOR, DEBUG_OUTPUT), not a 5-level chain.
2. **Feature Adaptation is Normal** — rename/redefine/undefine/select are standard MI tools, not code smells.
3. **Every Parent Has a Role** — One phrase: "traversal", "comparison", "specification", "debug output". Cannot name the role → don't add the parent.
4. **MI is for Composition** — Distinct capabilities combined, not code reuse (use `/eiffel.abstract` for reuse).
5. **Cursor Separates Concern** — Iteration is delegated to ITERATION_CURSOR, not built into structure.

## Feature Adaptation Quick Reference

| Strategy | When to Use | Example |
|----------|-------------|---------|
| `rename` | Semantic name conflict — same name, different meanings | `rename out as tagged_out end` |
| `redefine` | Descendant needs its own version | `redefine is_equal end` |
| `undefine` | Remove implementation from one parent (keep from other) | `undefine out end` |
| `select` | Repeated inheritance — choose which version for dynamic dispatch | `select is_equal end` |

**Key rules:**
- `rename` is for SEMANTIC conflicts, not compilation convenience
- `undefine` removes unwanted IMPLEMENTATIONS, not inconvenient features
- `select` is RARE — only for repeated inheritance with dynamic dispatch
- Always resolve `out` conflicts explicitly (most common MI conflict)

## Workflow

### Step 1: Identify Target

**If `design-scan.md` exists:** Read `<library-path>/.eiffel-workflow/evidence/design-scan.md` and present MI findings to user. Ask which finding to implement.

**If no scan exists:** Ask user:
```
Which class needs additional parent(s)?
What role should the new parent provide? (e.g., "iteration", "comparison", "serialization")
```

### Step 2: Analyze Existing Class

Read the target class thoroughly. Document:

1. **Current parents** — all classes in inherit clause, with any existing adaptation
2. **All features** — organized by category (Access, Status report, Commands, etc.)
3. **Existing conflicts** — features already using rename/redefine/undefine
4. **Feature names** that overlap with proposed parent(s)

**Also read the proposed parent class** (use Task/Explore agent if it's an ISE class):
```
Task(subagent_type=Explore) → "Show all features of ITERABLE including new_cursor signature"
```

### Step 3: Design MI Structure

For each new parent, produce:

**Role Justification** (mandatory — one phrase):
```
Parent: ITERABLE [G]
Role: "traversal" — enables across loop iteration over internal collection
```

**Conflict Resolution Table** (mandatory for every name clash):

| Feature | Existing Source | New Parent | Strategy | Rationale |
|---------|---------------|------------|----------|-----------|
| `out` | Current class | ITERABLE (from ANY) | `undefine` on ITERABLE | Existing `out` is more specific |
| `is_equal` | Current class | COMPARABLE | `redefine` | Need domain-specific equality |
| `new_cursor` | (none) | ITERABLE | implement | New feature required by parent |

**New Features Required:**
- List every deferred feature from the new parent that must be effected
- For ITERABLE: `new_cursor: ITERATION_CURSOR [G]` (must create and return a cursor)
- For COMPARABLE: `is_less alias "<" (other: like Current): BOOLEAN`
- For HASHABLE: `hash_code: INTEGER`

**Resulting Inherit Clause Preview:**
```eiffel
inherit
    EXISTING_PARENT
        -- existing adaptations preserved
    NEW_PARENT
        rename
            conflicting_name as new_parent_conflicting_name
        undefine
            out, is_equal, copy  -- common ANY features to undefine
        redefine
            -- features needing custom implementation
        end
```

### Step 4: Present Design for Approval

Show the user:
1. **Target class** and current parents
2. **Proposed parent(s)** with role justification
3. **Conflict resolution table** — every conflict and resolution strategy
4. **New features to implement** — what the class must now provide
5. **Resulting inherit clause** — full clause with all adaptation

**BLOCK until user approves.** Ask: "Proceed with this MI application? (yes/no/modify)"

### Step 5: Implement

**Oracle Gate:** Before writing any Eiffel code, check the oracle:
```bash
echo "$CODE" | /d/prod/simple_oracle/oracle-cli.exe check-code --stdin --library <library-name>
```

If oracle flags (exit 1): regenerate code addressing the flagged pattern. Repeat until exit 0.

**Implementation steps:**

1. **Update inherit clause** on target class with all adaptation (rename/redefine/undefine/select)
2. **Create role classes** if needed (e.g., a custom ITERATION_CURSOR descendant)
3. **Implement required features:**
   - For ITERABLE: create `new_cursor` returning appropriate ITERATION_CURSOR
   - For COMPARABLE: implement `is_less` with proper semantics
   - For HASHABLE: implement `hash_code` with proper distribution
4. **Add contracts** to new features (require/ensure as appropriate)
5. **Update ECF** if new dependency needed (e.g., adding simple_mml for MML_MODEL)

**ITERABLE Implementation Pattern** (most common MI addition):
```eiffel
-- In target class:
feature -- Iteration

    new_cursor: ITERATION_CURSOR [ELEMENT_TYPE]
            -- Fresh cursor for traversal.
        do
            Result := internal_collection.new_cursor
        ensure then
            cursor_exists: Result /= Void
        end

-- If internal collection's cursor doesn't match, create custom cursor:
-- src/my_class_cursor.e
class MY_CLASS_CURSOR

inherit
    ITERATION_CURSOR [ELEMENT_TYPE]

create
    make

feature {NONE} -- Initialization

    make (a_source: MY_CLASS)
        do
            source := a_source
            -- initialize cursor state
        end

feature -- Access

    item: ELEMENT_TYPE
        do
            Result := -- current element
        end

feature -- Status report

    after: BOOLEAN
        do
            Result := -- past end?
        end

feature -- Cursor movement

    forth
        do
            -- advance
        end

feature {NONE} -- Implementation

    source: MY_CLASS

end
```

### Step 6: Compile Gate

**CRITICAL: cd to the library directory before compiling.**

```bash
cd <library-path> && /d/prod/ec.sh -batch -config <library>.ecf -target <library>_tests -c_compile
```

**MUST see "System Recompiled."**

**Common MI compilation errors and fixes:**

| Error | Meaning | Fix |
|-------|---------|-----|
| **VMRC** | Feature from multiple parents, ambiguous | `undefine` on one parent |
| **VMFN** | Same feature name, different parents | `rename` on one parent or `undefine` |
| **VDRD** | Invalid redefinition (signature mismatch) | Fix return type conformance |
| **VLEL** | Select needed for dynamic dispatch | Add `select` clause (repeated inheritance only) |
| **VHPR** | Inheritance cycle detected | Restructure — cannot have circular inheritance |
| **ECMA-VMRC** | Conflicting effective features | `undefine` one, keep the other |

**Strategy for resolving errors:**
1. Read the FULL error message — it names the conflicting features and parents
2. For VMRC/VMFN: `undefine` the less specific version (usually from ANY via new parent)
3. For VDRD: ensure redefined feature's return type CONFORMS to parent's
4. Recompile after each fix — don't batch multiple guesses

**ZERO WARNINGS POLICY:** Fix all warnings immediately.

If compilation fails:
1. Record failure to oracle: `oracle-cli.exe record-failure compile <library> error.txt`
2. Fix the error
3. Recompile until success

### Step 7: Test + Evidence

**Run existing tests:**
```bash
cd <library-path> && ./EIFGENs/<library>_tests/W_code/<library>.exe
```

**ALL existing tests must pass.** MI application must preserve behavior.

**Save evidence** to `<library-path>/.eiffel-workflow/evidence/mi-application.txt`:

```
# MI Application Evidence
# Library: <library-path>
# Date: [timestamp]

## Target Class
Name: TARGET_CLASS
File: src/target_class.e

## Parents Added
1. ITERABLE [ELEMENT_TYPE]
   Role: "traversal"
   Justification: Enables across loop iteration over internal collection

2. COMPARABLE
   Role: "ordering"
   Justification: Natural ordering needed for sorted collections

## Conflict Resolution
| Feature | Existing | New Parent | Strategy | Rationale |
|---------|----------|------------|----------|-----------|
| out | keep | undefine | Existing out is domain-specific |
| is_equal | keep | undefine | Existing equality semantics correct |

## New Features Implemented
  - new_cursor: ITERATION_CURSOR [ELEMENT_TYPE]
    Contract: ensure cursor_exists: Result /= Void
  - is_less (other: like Current): BOOLEAN
    Contract: require other_exists: other /= Void

## Role Classes Created
  - TARGET_CLASS_CURSOR (src/target_class_cursor.e)
    Inherits: ITERATION_CURSOR [ELEMENT_TYPE]

## Compilation
Command: /d/prod/ec.sh -batch -config <lib>.ecf -target <lib>_tests -c_compile
Status: PASS
Warnings: NONE
MI errors resolved: [list or NONE]

## Tests
Command: ./EIFGENs/<lib>_tests/W_code/<lib>.exe
Status: ALL PASS ([n]/[n])
```

## Completion

```
MI application COMPLETE for: <library-path>

Target: TARGET_CLASS
  - Parents added: [list with roles]
  - Conflicts resolved: [n]
  - New features: [n]
  - Role classes created: [list or none]
  - Compilation: PASS
  - Tests: ALL PASS ([n]/[n])

Evidence: <library-path>/.eiffel-workflow/evidence/mi-application.txt
```

**Do NOT suggest next skills automatically.** The MI work is done — let the user decide what's next. Only mention other skills if the user asks.

## Context Management (RLM Pattern)

This skill focuses ONLY on: `<library-path>`

**DO NOT:**
- Read files outside this library directory
- Load entire ecosystem into context
- Keep large file contents in working memory after analysis

**DO:**
- Use Task tool with Explore agent for ISE class API questions
- Ask targeted questions: "What features does ITERABLE require descendants to implement?"
- Release context after getting answers

## Anti-Drift

- **Every parent MUST have a documented ROLE.** Cannot name the role in one phrase → don't add the parent.
- **`undefine` removes unwanted implementations, NOT inconvenient features.** If a feature is semantically needed, don't undefine it.
- **Always resolve `out` conflicts explicitly.** It's the most common source of VMRC errors.
- **MI application must PRESERVE all existing tests.** Zero regressions.
- **Oracle Gate before writing Eiffel code.** No exceptions.
- **Skill Version Lock:** If you discover skill improvements during workflow, queue them in `<library-path>/.eiffel-workflow/skill-improvements.md` — do NOT modify skills mid-workflow
