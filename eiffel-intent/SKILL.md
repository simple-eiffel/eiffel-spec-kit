---
name: eiffel-intent
description: Phase 0 of Eiffel Spec Kit. Generates intent.md capturing WHAT users need and WHY. Runs AI-assisted intent review with probing questions. Use with /eiffel.intent command.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Task
---

# /eiffel.intent - Phase 0: Intent Capture

**Purpose:** Capture WHAT users need and WHY before any code is written.

## Usage

```
/eiffel.intent <project-path>
```

**Example:**
```
/eiffel.intent d:\prod\simple_cache
```

If no path provided, ask user: "Which project? Provide the full path (e.g., d:\prod\simple_cache)"

## Project Scoping

All files are created inside the PROJECT directory:
```
<project-path>/
├── .eiffel-workflow/
│   ├── intent.md
│   ├── intent-v2.md (after review)
│   ├── prompts/
│   │   └── phase0-intent-review.md  (prompt for external AI)
│   └── evidence/
│       └── phase0-intent.txt
├── src/
├── test/
└── <project>.ecf
```

## Workflow

### Step 1: Create Directory Structure

```bash
mkdir -p <project-path>/.eiffel-workflow/prompts
mkdir -p <project-path>/.eiffel-workflow/evidence
```

### Step 2: Generate Initial intent.md

Ask the user to describe what they want to build. Create `<project-path>/.eiffel-workflow/intent.md`:

```markdown
# Intent: [Library/Feature Name]

## What
[Clear description of what this does]

## Why
[Business/technical reason this is needed]

## Users
[Who will use this and how]

## Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] [Criterion 3]

## Out of Scope
[What this explicitly does NOT do]

## Dependencies
[Other simple_* libraries or systems required]
```

### Step 3: Generate AI Review Prompt File (MANUAL CYCLE)

Create `<project-path>/.eiffel-workflow/prompts/phase0-intent-review.md`:

```markdown
# Intent Review Request

**Instructions:** Review the intent document below and generate probing questions
to clarify vague language, identify missing requirements, and surface implicit assumptions.

## Review Criteria

Look for:
1. **Vague language:** Words like "fast", "secure", "easy", "flexible" without concrete definitions
2. **Missing edge cases:** What happens with empty input? Maximum size? Invalid data?
3. **Untestable criteria:** Are acceptance criteria specific and measurable?
4. **Hidden dependencies:** What external systems or libraries are assumed?
5. **Scope ambiguity:** Is "out of scope" clearly defined?

## Output Format

Provide 5-10 probing questions. For each:
- Quote the vague phrase
- Explain why it's vague
- Offer 2-3 concrete alternatives the user can choose from

---

## Intent Document to Review

[PASTE CONTENTS OF intent.md HERE]
```

### Step 3b: Auto-Create Evidence Placeholder File

Create `<project-path>/.eiffel-workflow/evidence/phase0-ai-review.md` with placeholder:

```markdown
# Phase 0: AI Review Response

**STATUS: INCOMPLETE** - Paste AI review response below this line and delete this section.

---

## Instructions

1. Open: `../.eiffel-workflow/prompts/phase0-intent-review.md`
2. Paste intent.md contents where indicated
3. Submit to an external AI (Ollama, Grok, Gemini, or another Claude session)
4. Replace this entire file with the AI's response
5. Return to Claude Code and say "review complete"

---

[AI RESPONSE GOES HERE]
```

### Step 4: Notify Human

Display:
```
PHASE 0: Intent document created.

FILES CREATED:
  - <project-path>/.eiffel-workflow/intent.md
  - <project-path>/.eiffel-workflow/prompts/phase0-intent-review.md
  - <project-path>/.eiffel-workflow/evidence/phase0-ai-review.md (placeholder - needs your input)

MANUAL ACTION REQUIRED:
  1. Open: <project-path>/.eiffel-workflow/prompts/phase0-intent-review.md
  2. Copy the intent.md content into the prompt where indicated
  3. Submit to an AI reviewer (Ollama, Grok, Gemini, or another Claude session)
  4. Replace contents of: <project-path>/.eiffel-workflow/evidence/phase0-ai-review.md with AI's response
  5. Return here and say "review complete" with any answers to the questions

Waiting for your input...
```

**BLOCK until user provides the review results.**

### Step 5: Process Review Results

When user returns with AI review:
1. Read the probing questions from the AI
2. Ask user to answer each question
3. Create `<project-path>/.eiffel-workflow/intent-v2.md` incorporating answers

### Step 6: Final Approval

Ask: **"Intent document refined. Approve to proceed to Phase 1 (Contracts)?"**

**BLOCK until user explicitly approves.**

### Step 7: Save Evidence

Save to `<project-path>/.eiffel-workflow/evidence/phase0-intent.txt`:
```
# Phase 0 Intent Evidence
# Project: <project-path>
# Date: [timestamp]

Intent document: .eiffel-workflow/intent-v2.md
AI review conducted: [yes/no]
AI used for review: [Ollama/Grok/Gemini/Claude/other]
Questions answered: [count]
User approved: [yes/no]

# Status: APPROVED
```

## Completion

When user approves:
```
Phase 0 COMPLETE: Intent captured and approved.
Project: <project-path>

Next: Run /eiffel.contracts <project-path> to generate contract skeletons.
```

## Context Management (RLM Pattern)

This skill focuses ONLY on: `<project-path>`

**DO NOT:**
- Read files outside this project directory
- Load entire ecosystem or multiple libraries into context
- Keep large file contents in working memory

**DO:**
- Use Task tool with Explore agent for ecosystem questions
- Ask targeted questions: "What does simple_mml provide?" not "Show me all files"
- Release context after getting answers

**Example - Ecosystem Query:**
```
Need to know about a dependency? Don't read all its files.
Instead: Task(subagent_type=Explore) → "What are simple_mml's main classes and key features?"
```

The sub-agent searches, summarizes, and returns ONLY what you need. Your context stays focused on this project.

## Anti-Drift

- Prompt files are artifacts (can't claim review happened without them)
- Human manually submits to AI (forced engagement)
- Evidence file documents what AI was used
