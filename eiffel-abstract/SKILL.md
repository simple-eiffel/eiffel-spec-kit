---
name: eiffel-abstract
description: Extract shared behavior into deferred class abstractions following ISE EiffelBase patterns. Each abstraction level adds contracts, not implementation. Use with /eiffel.abstract command.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task
---

# /eiffel.abstract - Deferred Class Extraction

**Purpose:** Extract shared behavior from 2+ concrete classes into a deferred class abstraction following ISE EiffelBase patterns. Each abstraction level adds contracts, not implementation.

## Usage

```
/eiffel.abstract <library-path>
```

**Example:**
```
/eiffel.abstract d:\prod\simple_sorter
```

If no path provided, ask user: "Which library? Provide the full path (e.g., d:\prod\simple_sorter)"

## Project Scoping

```
<library-path>/
├── .eiffel-workflow/
│   └── evidence/
│       ├── design-scan.md           ← INPUT (optional, from /eiffel.design-scan)
│       └── abstract-extraction.txt  ← OUTPUT (evidence)
├── src/
│   └── *.e                          ← MODIFIED (add deferred class, update descendants)
├── test/
│   └── *.e                          ← VERIFIED (existing tests must pass)
└── <library>.ecf                    ← MAY UPDATE (if new file added)
```

## Design Ethics (ISE EiffelBase Reference)

1. **Deferred as Contract** — Abstract classes define WHAT via contracts, not HOW. Postconditions are mandatory on deferred features.
2. **One Deferred Feature = One Responsibility** — Each deferred feature represents exactly one decision a descendant must make.
3. **Template Method** — Make ONE feature deferred, derive others from it. Reference: PART_COMPARABLE where `<` generates `<=`, `>`, `>=`.
4. **Invariants Enforce Laws** — Class invariants constrain all descendants uniformly.
5. **Flat-and-Wide over Deep** — Do not create chains >3 levels without user approval.

## Workflow

### Step 1: Identify Target

**If `design-scan.md` exists:** Read `<library-path>/.eiffel-workflow/evidence/design-scan.md` and present abstraction findings to user. Ask which finding to implement.

**If no scan exists:** Ask user:
```
Which classes share behavior that should be extracted?
Provide 2+ class names (e.g., BUBBLE_SORT, QUICK_SORT)
```

### Step 2: Analyze Shared Features

Read all target classes. For each feature that appears in multiple classes, classify:

| Feature | Contract | Implementation | Action |
|---------|----------|---------------|--------|
| Same name+sig | Same pre/post | Same body | **EFFECTIVE** in ancestor |
| Same name+sig | Same pre/post | Different body | **DEFERRED** with contract |
| Same name+sig | Different pre/post | Different body | **Cannot extract** (flag for review) |
| Unique to one class | — | — | **Stays in descendant** |

**Mandatory checks:**
- Count features in each category
- Identify Template Method candidates (one feature generating others)
- Identify shared invariant conditions across classes
- Identify creation procedure patterns (same make signature?)

### Step 3: Design the Deferred Class

Create a class skeleton with:

```eiffel
deferred class
    DEFERRED_NAME [G -> CONSTRAINT]  -- generic only if needed

feature -- Access

    shared_query: RETURN_TYPE
            -- Description from shared contract.
        deferred
        ensure
            result_property: Result.some_property  -- MANDATORY postcondition
        end

feature -- Status report

    is_valid: BOOLEAN
            -- Description.
        deferred
        ensure
            definition: Result = (some_condition)  -- MANDATORY postcondition
        end

feature -- Basic operations

    shared_effective_operation
            -- Shared implementation (same in all descendants).
        require
            valid_state: is_valid
        do
            -- Actual shared implementation code
        ensure
            state_changed: some_postcondition
        end

    -- Template Method pattern
    derived_operation: BOOLEAN
            -- Derived from `base_operation`.
        do
            Result := not base_operation  -- effective, derived from deferred
        ensure
            definition: Result = not base_operation
        end

    base_operation: BOOLEAN
            -- Override point for Template Method.
        deferred
        ensure
            -- postcondition on the one deferred drives the others
        end

invariant
    shared_law: some_condition_all_descendants_must_satisfy

end
```

**CRITICAL RULES:**
- Every deferred feature MUST have a postcondition. A deferred feature without a postcondition is a specification gap — flag it to the user.
- Effective features in the ancestor must have IDENTICAL implementation across all source classes. If they differ even slightly, make it DEFERRED.
- Prefer Template Method: make ONE feature deferred, derive others from it.

### Step 4: Present Design for Approval

Show the user:
1. **Proposed deferred class** — full skeleton
2. **Descendant changes** — what each concrete class gains/loses:
   ```
   CONCRETE_A:
     + inherits DEFERRED_NAME
     - removes: shared_effective_operation (now inherited)
     + provides: base_operation (effecting deferred)

   CONCRETE_B:
     + inherits DEFERRED_NAME
     - removes: shared_effective_operation (now inherited)
     + provides: base_operation (effecting deferred)
   ```
3. **Hierarchy depth** — resulting chain depth (warn if >3)
4. **Test impact** — which tests cover affected features

**BLOCK until user approves.** Ask: "Proceed with this extraction? (yes/no/modify)"

### Step 5: Implement

**Oracle Gate:** Before writing any Eiffel code, check the oracle:
```bash
echo "$CODE" | /d/prod/simple_oracle/oracle-cli.exe check-code --stdin --library <library-name>
```

If oracle flags (exit 1): regenerate code addressing the flagged pattern. Repeat until exit 0.

**Implementation steps:**
1. Create deferred class file in `<library-path>/src/`
2. Update each descendant's inherit clause:
   ```eiffel
   inherit
       DEFERRED_NAME
           redefine
               -- any features that need descendant-specific behavior
           end
   ```
3. Remove features from descendants that are now inherited (effective from ancestor)
4. Ensure descendants provide effective versions of all deferred features
5. Update ECF if new source file added (cluster should auto-pick up `.e` files in `src/`)

### Step 6: Compile Gate

**CRITICAL: cd to the library directory before compiling.**

```bash
cd <library-path> && /d/prod/ec.sh -batch -config <library>.ecf -target <library>_tests -c_compile
```

**MUST see "System Recompiled."**

**Common errors and fixes:**
- **VDRD** (invalid redefinition): Check signature matches — return type must conform, not be identical
- **VDJR** (join rule): Effective feature in descendant must match deferred signature exactly
- **VCFG** (constraint violation): Generic parameter constraint mismatch
- **VTCT** (unknown class): New deferred class not in cluster path — check ECF

**ZERO WARNINGS POLICY:** Fix all warnings immediately before proceeding.

If compilation fails:
1. Record failure to oracle: `oracle-cli.exe record-failure compile <library> error.txt`
2. Fix the error
3. Recompile until success

### Step 7: Test + Evidence

**Run existing tests:**
```bash
cd <library-path> && ./EIFGENs/<library>_tests/W_code/<library>.exe
```

**ALL existing tests must pass.** Extraction must preserve behavior.

**Save evidence** to `<library-path>/.eiffel-workflow/evidence/abstract-extraction.txt`:

```
# Abstract Extraction Evidence
# Library: <library-path>
# Date: [timestamp]

## Deferred Class Created
Name: DEFERRED_NAME
File: src/deferred_name.e

## Features Extracted
Deferred (contract only):
  - feature_a: RETURN_TYPE (postcondition: ...)
  - feature_b (postcondition: ...)

Effective (shared implementation):
  - shared_op: moved from [CLASS_A, CLASS_B]

Template Methods:
  - derived_op: derived from feature_a

## Descendants Updated
  - CLASS_A: added inherit DEFERRED_NAME, removed shared_op, effects feature_a
  - CLASS_B: added inherit DEFERRED_NAME, removed shared_op, effects feature_b

## Invariants
  - shared_law: [description]

## Compilation
Command: /d/prod/ec.sh -batch -config <lib>.ecf -target <lib>_tests -c_compile
Status: PASS
Warnings: NONE

## Tests
Command: ./EIFGENs/<lib>_tests/W_code/<lib>.exe
Status: ALL PASS ([n]/[n])
```

## Completion

```
Abstract extraction COMPLETE for: <library-path>

Deferred class: DEFERRED_NAME
  - Deferred features: [n] (all with postconditions)
  - Effective features: [n] (shared implementations)
  - Template Methods: [n]
  - Descendants updated: [list]
  - Compilation: PASS
  - Tests: ALL PASS ([n]/[n])

Evidence: <library-path>/.eiffel-workflow/evidence/abstract-extraction.txt
```

**Do NOT suggest next skills automatically.** The extraction is done — let the user decide what's next. Only mention other skills if the user asks.

## Context Management (RLM Pattern)

This skill focuses ONLY on: `<library-path>`

**DO NOT:**
- Read files outside this library directory
- Load entire ecosystem into context
- Keep large file contents in working memory after analysis

**DO:**
- Use Task tool with Explore agent for ISE EiffelBase pattern questions
- Ask targeted questions: "How does PART_COMPARABLE implement Template Method?"
- Release context after getting answers

## Anti-Drift

- **NEVER move different implementations into the ancestor as effective.** If descendants differ, make it DEFERRED with contracts.
- **ALWAYS define postconditions on deferred features.** Flag specification gaps to the user.
- **Extraction must PRESERVE all existing tests.** Zero regressions.
- **Do not create chains >3 levels without user approval.**
- **Oracle Gate before writing Eiffel code.** No exceptions.
- **Skill Version Lock:** If you discover skill improvements during workflow, queue them in `<library-path>/.eiffel-workflow/skill-improvements.md` — do NOT modify skills mid-workflow
