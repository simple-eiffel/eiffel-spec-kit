# Eiffel Spec Kit

**Claude Code skills for Design by Contract development workflow**

## Overview

The Eiffel Spec Kit is a set of Claude Code skills that enforce a disciplined 8-phase development workflow for Eiffel libraries. Each phase produces evidence files that create an audit trail from intent to ship.

## Phases

| Phase | Skill | Purpose |
|-------|-------|---------|
| 0 | `/eiffel.intent` | Capture WHAT and WHY before coding |
| 1 | `/eiffel.contracts` | Generate class skeletons with DBC contracts |
| 2 | `/eiffel.review` | Multi-AI adversarial review chain |
| 3 | `/eiffel.tasks` | Break contracts into implementation tasks |
| 4 | `/eiffel.implement` | Write feature bodies (contracts FROZEN) |
| 5 | `/eiffel.verify` | Generate and run test suite |
| 6 | `/eiffel.harden` | MML model queries + adversarial tests |
| 7 | `/eiffel.ship` | Final checklist and human approval |

## Supporting Skills

| Skill | Purpose |
|-------|---------|
| `/eiffel.status` | Show current phase and evidence |
| `/eiffel.expert` | Eiffel programming expertise |

## Key Principles

1. **Contracts ARE Tests** - Preconditions, postconditions, and invariants brought into the class where they have semantic meaning
2. **Progressive AI Review** - Ollama → Claude → Grok → Gemini → Human chain catches blind spots
3. **Evidence Trail** - Every phase produces `.eiffel-workflow/evidence/` files
4. **Human Gates** - Phases 2, 7 block until explicit human approval
5. **MML Integration** - Mathematical Model Library for precise postconditions

## First Successful Run

**proven_fetch v1.0.0** - Built using all 8 phases in 2h18m (2026-01-22)
- 5 classes, 53 tests (30 unit + 23 adversarial)
- MML model queries with frame conditions
- SCOOP-compatible, void-safe

## Installation

These skills are designed for use with Claude Code. Place the skill folders in your Claude Code skills directory.

## License

MIT License

---

Part of the [Simple Eiffel](https://github.com/simple-eiffel) ecosystem.
