---
name: eiffel-generify
description: Convert concrete-typed Eiffel classes to use generic parameters with proper constraints following ISE EiffelBase patterns. Use with /eiffel.generify command.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task
---

# /eiffel.generify - Generic Parameterization

**Purpose:** Convert concrete-typed classes to use generic parameters with proper constraints following ISE EiffelBase patterns. Constraints are design decisions, not afterthoughts.

## Usage

```
/eiffel.generify <library-path>
```

**Example:**
```
/eiffel.generify d:\prod\simple_sorter
```

If no path provided, ask user: "Which library? Provide the full path (e.g., d:\prod\simple_sorter)"

## Project Scoping

```
<library-path>/
├── .eiffel-workflow/
│   └── evidence/
│       ├── design-scan.md         ← INPUT (optional, from /eiffel.design-scan)
│       └── generification.txt     ← OUTPUT (evidence)
├── src/
│   └── *.e                        ← MODIFIED (class headers, feature signatures)
├── test/
│   └── *.e                        ← MODIFIED (update type instantiations)
└── <library>.ecf                  ← MAY UPDATE (if dependencies change)
```

## Design Ethics (ISE EiffelBase Reference)

1. **Generic Constraints Guide Implementation** — Constraints are design decisions. `[G -> COMPARABLE]` means "this class needs ordering". An unconstrained `[G]` means "this class truly works with ANY type."
2. **Constraint Determines Operations** — Every operation used on G must be justified by its constraint.
3. **like Current Preserves Type** — For binary operations where both operands and result share the same type.
4. **SCOOP Compatibility** — Include `detachable separate` when types cross processor boundaries.
5. **One Parameter = One Design Decision** — Each generic parameter represents exactly one type choice the client makes.

## Constraint Reference Table

| Operation Used on G | Required Constraint | ISE Example |
|--------------------|-------------------|-------------|
| `<` `>` `<=` `>=` | `G -> COMPARABLE` | `SORTED_LIST [G -> COMPARABLE]` |
| `hash_code` | `G -> HASHABLE` | `HASH_TABLE [G, K -> HASHABLE]` |
| `=` `/=` `~` (object equality) | (none needed — available on ANY) | `ARRAYED_LIST [G]` |
| `+` `-` `*` `/` (arithmetic) | `G -> NUMERIC` | `MATRIX [G -> NUMERIC]` |
| binary ops returning same type | `like Current` pattern | `NUMERIC` descendants |
| SCOOP safety | `G -> detachable separate ANY` | `MML_MAP [K -> detachable separate ANY, V -> detachable separate ANY]` |
| creation | `G -> CREATABLE create make end` | `FACTORY [G -> CREATABLE create make end]` |
| domain-specific | `G -> SPECIFIC_CLASS` | `SIMPLE_FACTORY [G -> SIMPLE_CREATABLE]` |

## Workflow

### Step 1: Identify Target

**If `design-scan.md` exists:** Read `<library-path>/.eiffel-workflow/evidence/design-scan.md` and present generics findings to user. Ask which finding to implement.

**If no scan exists:** Ask user:
```
Which class should be parameterized?
Which concrete type(s) should become generic parameter(s)?
```

### Step 2: Analyze Type Dependencies

Read the target class. For every feature that uses the concrete type, record:

1. **Feature name and signature** — where the type appears (parameter, return, local)
2. **Operations performed** — what is called on instances of this type
3. **Creation patterns** — is the type created within the class? How?
4. **Cross-references** — other classes that use this type from the target class

Build a complete map:

```
Type: STRING (candidate for G)

Features using STRING:
  - put (a_key: STRING; a_value: INTEGER)     [parameter]
  - item (a_key: STRING): INTEGER              [parameter]
  - has_key (a_key: STRING): BOOLEAN           [parameter]
  - keys: ARRAYED_LIST [STRING]                [return type]
  - internal_table: HASH_TABLE [INTEGER, STRING]  [attribute]

Operations on STRING:
  - is_equal (via `~` in has_key)              → no constraint needed
  - hash_code (via HASH_TABLE key)             → G -> HASHABLE
  - out (in debug output)                      → no constraint needed (from ANY)

Constraint determination: G -> HASHABLE
```

### Step 3: Design Generic Parameters

For each parameter, specify:

| Parameter | Replaces | Constraint | Justification |
|-----------|----------|-----------|---------------|
| `K` | STRING (key type) | `K -> HASHABLE` | Used as HASH_TABLE key, needs hash_code |
| `V` | INTEGER (value type) | (none) | Only stored and retrieved, no operations |

**Naming conventions:**
- `G` — single general parameter
- `K` — key type
- `V` — value type
- `G -> COMPARABLE` — when ordering is needed
- `G -> HASHABLE` — when hashing is needed

**SCOOP check:**
- If the library's ECF has `<concurrency support="scoop"/>`, consider whether parameters need `detachable separate`
- For public API types that clients may pass across SCOOP processors: use `detachable separate` constraints
- For internal-only types: standard constraints are fine

**like Current check:**
- If binary operations exist where both operands and result should be the same type, use `like Current` instead of a generic parameter
- Example: `plus (other: like Current): like Current` for arithmetic types

**Creation constraint check:**
- If the class creates instances of G internally, a creation constraint is needed:
  `G -> SOME_CLASS create make end`
- If the class only receives G from outside, no creation constraint needed

### Step 4: Present Design for Approval

Show the user:

1. **Proposed class header:**
   ```eiffel
   class MY_CLASS [K -> HASHABLE, V]
   ```

2. **Feature signature changes** (before → after):
   ```
   put (a_key: STRING; a_value: INTEGER)  →  put (a_key: K; a_value: V)
   item (a_key: STRING): INTEGER          →  item (a_key: K): V
   keys: ARRAYED_LIST [STRING]            →  keys: ARRAYED_LIST [K]
   ```

3. **Attribute type changes:**
   ```
   internal_table: HASH_TABLE [INTEGER, STRING]  →  HASH_TABLE [V, K]
   ```

4. **Client code impact** (files in src/ AND test/ that instantiate the class):
   ```
   test/test_my_class.e: line 15: create l_obj.make  →  l_obj: MY_CLASS [STRING, INTEGER]; create l_obj.make
   src/other_class.e: line 42: my_ref: MY_CLASS      →  my_ref: MY_CLASS [STRING, INTEGER]
   ```

5. **Constraint justification** — for each constraint, which operations require it

**BLOCK until user approves.** Ask: "Proceed with this generification? (yes/no/modify)"

### Step 5: Implement

**Oracle Gate:** Before writing any Eiffel code, check the oracle:
```bash
echo "$CODE" | /d/prod/simple_oracle/oracle-cli.exe check-code --stdin --library <library-name>
```

If oracle flags (exit 1): regenerate code addressing the flagged pattern. Repeat until exit 0.

**Implementation steps:**

1. **Update class header** — add generic parameters with constraints
2. **Replace concrete types in features:**
   - Parameter types
   - Return types
   - Local variable types
   - Attribute types
3. **Update creation procedures** — ensure they work with generic types
4. **Update ALL client code** — search BOTH `src/` AND `test/`:
   ```bash
   # Find all references to the class name
   grep -rn "MY_CLASS" <library-path>/src/ <library-path>/test/
   ```
   Update every instantiation to provide type parameters:
   ```eiffel
   -- Before:
   l_obj: MY_CLASS
   -- After:
   l_obj: MY_CLASS [STRING, INTEGER]
   ```
5. **Update inherit clauses** in any descendants:
   ```eiffel
   -- Before:
   inherit MY_CLASS
   -- After:
   inherit MY_CLASS [STRING, INTEGER]  -- or propagate generic parameter
   ```

**CRITICAL:** Do not forget test files. Grep for the class name across the ENTIRE library.

### Step 6: Compile Gate

**CRITICAL: cd to the library directory before compiling.**

```bash
cd <library-path> && /d/prod/ec.sh -batch -config <library>.ecf -target <library>_tests -c_compile
```

**MUST see "System Recompiled."**

**Common generification compilation errors and fixes:**

| Error | Meaning | Fix |
|-------|---------|-----|
| **VTCG** | Client code violates constraint | Client must provide type that conforms to constraint |
| **VTCT** | Generic parameter name conflicts with class name | Rename parameter (G, K, V, etc.) |
| **VUAR** | Operation on generic requires constraint | Add appropriate constraint to parameter |
| **VEEN** | Missing type parameter in client code | Add `[TYPE]` to all class references |
| **VDRD** | Redefined feature signature doesn't match | Update descendants to use generic parameter |
| **VTUG** | Wrong number of generic parameters | Ensure all references have correct parameter count |

**Strategy for resolving errors:**
1. Start with VTCG/VEEN — these are client code that forgot type parameters
2. Then VUAR — operations need stronger constraints
3. Then VDRD/VTUG — inheritance and descendant issues
4. Recompile after each fix

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

**ALL existing tests must pass.** Generification must preserve behavior.

**Save evidence** to `<library-path>/.eiffel-workflow/evidence/generification.txt`:

```
# Generification Evidence
# Library: <library-path>
# Date: [timestamp]

## Target Class
Name: MY_CLASS → MY_CLASS [K -> HASHABLE, V]
File: src/my_class.e

## Generic Parameters Added
1. K -> HASHABLE
   Replaces: STRING
   Justification: Used as HASH_TABLE key, requires hash_code
   Operations requiring constraint: hash_code (via HASH_TABLE), is_equal

2. V (unconstrained)
   Replaces: INTEGER
   Justification: Only stored and retrieved, no operations performed
   SCOOP note: No cross-processor usage detected

## Feature Signature Changes
  - put (a_key: STRING; a_value: INTEGER) → put (a_key: K; a_value: V)
  - item (a_key: STRING): INTEGER → item (a_key: K): V
  - has_key (a_key: STRING): BOOLEAN → has_key (a_key: K): BOOLEAN
  - keys: ARRAYED_LIST [STRING] → keys: ARRAYED_LIST [K]

## Attribute Changes
  - internal_table: HASH_TABLE [INTEGER, STRING] → HASH_TABLE [V, K]

## Client Code Updated
  - test/test_my_class.e: 3 references updated to MY_CLASS [STRING, INTEGER]
  - src/other_class.e: 1 reference updated to MY_CLASS [STRING, INTEGER]

## Compilation
Command: /d/prod/ec.sh -batch -config <lib>.ecf -target <lib>_tests -c_compile
Status: PASS
Warnings: NONE
Generification errors resolved: [list or NONE]

## Tests
Command: ./EIFGENs/<lib>_tests/W_code/<lib>.exe
Status: ALL PASS ([n]/[n])
```

## Completion

```
Generification COMPLETE for: <library-path>

Class: MY_CLASS → MY_CLASS [K -> HASHABLE, V]
  - Parameters added: [n] ([list with constraints])
  - Features updated: [n]
  - Client files updated: [n]
  - Compilation: PASS
  - Tests: ALL PASS ([n]/[n])

Evidence: <library-path>/.eiffel-workflow/evidence/generification.txt
```

**Do NOT suggest next skills automatically.** The generification is done — let the user decide what's next. Only mention other skills if the user asks.

## Context Management (RLM Pattern)

This skill focuses ONLY on: `<library-path>`

**DO NOT:**
- Read files outside this library directory
- Load entire ecosystem into context
- Keep large file contents in working memory after analysis

**DO:**
- Use Task tool with Explore agent for ISE EiffelBase generic pattern questions
- Ask targeted questions: "How is SORTED_LIST [G -> COMPARABLE] constrained?"
- Release context after getting answers

## Anti-Drift

- **NEVER add unconstrained generics unless the class truly works with ANY type.** If you use no operations on G, why is it generic?
- **ALWAYS search ALL files (src/ + test/) when updating client code.** Missing a reference causes VEEN errors.
- **Generification must PRESERVE all existing tests.** Zero regressions.
- **Constraint must be MINIMAL but SUFFICIENT.** Don't add COMPARABLE if only `is_equal` is used (that's from ANY).
- **Oracle Gate before writing Eiffel code.** No exceptions.
- **Skill Version Lock:** If you discover skill improvements during workflow, queue them in `<library-path>/.eiffel-workflow/skill-improvements.md` — do NOT modify skills mid-workflow
