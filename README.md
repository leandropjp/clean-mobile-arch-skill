# Clean Mobile Architecture — Claude Code Skills

Turn **"Clean Mobile Architecture"** by [Petros Efthymiou](https://petrosefthymiou.com/) into an interactive architecture advisor for your Android, iOS, and Flutter projects.

> "The only way to go fast is to go well." — Robert C. Martin

These skills encode the book's frameworks, principles, and decision trees into actionable Claude Code commands that review your code, diagnose architectural problems, and guide design decisions — channeling the author's philosophy directly into your workflow.

## Quick Start

### Install from GitHub

```bash
claude /install-plugin leandropjp/clean-mobile-arch-skill
```

### Or clone locally

```bash
git clone https://github.com/leandropjp/clean-mobile-arch-skill.git
cd clean-mobile-arch-skill
claude  # skills auto-detected
```

## Skills

| Command | What It Does |
|---|---|
| `/clean-arch` | Main advisor — routes to the right sub-skill based on your question |
| `/clean-arch review` | Review your code against Clean Architecture principles |
| `/clean-arch solid` | Check S.O.L.I.D principle compliance with red-flag checklists |
| `/clean-arch layers` | Structure Domain / Data / Presentation layers and boundaries |
| `/clean-arch pattern` | Choose between MVC, MVP, MVVM, and MVI for your context |
| `/clean-arch test` | Design testable architecture and apply TDD |
| `/clean-arch diagnose` | "What's wrong with my architecture?" — full diagnostic |

## What's Inside

The skills distill the book's 20 chapters into actionable frameworks:

### Core Concepts

- **MVA** (Minimum Viable Architecture) — the simplest architecture that is still clean. Evolves via preparatory refactoring.
- **SCP** (Single Concern Principle) — every class is exactly one of: Executor, Orchestrator, or Data Container.
- **FSO** (Fundamental Software Objectives) — code must be Readable, Testable, Extensible, Robust, Maintainable, Resilient.
- **Contain Cancer** — isolate legacy and third-party code behind boundaries. Contained cancer can be treated; spread cancer is fatal.

### Decision Frameworks

**MV\* Pattern Selection:**
```
Team knows Reactive Programming?
  NO  → MVP
  YES → App complexity?
          LOW/MEDIUM → MVVM
          HIGH       → MVI-like + Clean Architecture
```

**Boundary Strategy:**
| Context | Recommendation |
|---|---|
| Medium complexity | Soft boundaries (packages) |
| Enterprise / large team | Feature modules (hard) + layer packages (soft) |
| 50+ developers | Mobile microservices (hard feature + hard layer) |

**Architecture Scenarios:**
| | Enterprise | Startup |
|---|---|---|
| Layers | Domain → Application → Presentation & Data → Framework | Merged Application + Presentation |
| Boundaries | Hard (modules) | Soft (packages) |
| DIP | Full (DataSource + DataGateway) | Repository pattern |
| Testing | White-box unit tests per layer | Black-box across layers |

### S.O.L.I.D Checklists

Each principle comes with concrete red flags to check:

- **S** — Is this class mixing Executor + Orchestrator roles? Is a ViewModel serving multiple screens?
- **O** — Are there growing if/else blocks that should use Strategy pattern?
- **L** — Does a subclass impose stricter input validation than its parent?
- **I** — Are there bulky interfaces, null parameters, or empty overridden functions?
- **D** — Does the domain layer import from data layer concretions?

## How It Works

The skills don't summarize the book — they **apply** its frameworks to **your** code. When you run `/clean-arch review`, the advisor:

1. Asks about your context (platform, complexity, team size)
2. Reads your actual code
3. Classifies each class (Executor / Orchestrator / Data Container)
4. Checks against S.O.L.I.D checklists and the Dependency Rule
5. Assesses MVA fitness (over-engineered? under-engineered? stale?)
6. Delivers prioritized, actionable fixes

## Project Structure

```
SKILL.md                              # Main orchestrator + routing
references/
  core-principles.md                   # Book philosophy, mental models, key quotes
  solid-checklist.md                   # All 5 principles with red-flag checklists
  architecture-layers.md               # Clean Architecture layers, 7 logic types, boundaries
  pattern-decision-tree.md             # MV* comparison and decision flow
  testing-strategy.md                  # Testing pyramid, TDD schools, strategies
skills/
  clean-arch-review/SKILL.md           # Architecture review
  clean-arch-solid/SKILL.md            # S.O.L.I.D compliance
  clean-arch-layers/SKILL.md           # Layer structuring
  clean-arch-pattern/SKILL.md          # MV* pattern selection
  clean-arch-test/SKILL.md             # Testing and TDD
  clean-arch-diagnose/SKILL.md         # Architecture diagnosis
```

## Inspired By

This project follows the "book-as-a-skill" pattern pioneered by [Sahil Lavingia's skills](https://github.com/slavingia/skills), which turned *The Minimalist Entrepreneur* into Claude Code skills.

The PDF-to-Markdown conversion was done with [Docling](https://github.com/docling-project/docling) using the VLM pipeline with MLX acceleration on Apple Silicon.

## About the Book

**Clean Mobile Architecture** by Petros Efthymiou (2022) covers:
- **Part I** — Why clean architecture matters (technical debt, MVA, effectiveness)
- **Part II** — How to write clean code (S.O.L.I.D, Reactive Programming, IoC, TDD)
- **Part III** — What clean mobile architecture looks like (Clean Architecture, MV\* patterns, layers, boundaries)

The book is platform-agnostic with examples in Kotlin (Android) and Swift (iOS).

[Get the book](https://www.amazon.com/dp/B0BFZF93QN) to dive deeper into the theory, case studies, and code samples.

## License

MIT
