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
| `/clean-arch team` | **Start here** — Profile your team for tailored architecture advice |
| `/clean-arch` | Main advisor — routes to the right sub-skill based on your question |
| `/clean-arch review` | Review your code against Clean Architecture principles |
| `/clean-arch solid` | Check S.O.L.I.D principle compliance with red-flag checklists |
| `/clean-arch layers` | Structure Domain / Data / Presentation layers and boundaries |
| `/clean-arch pattern` | Choose between MVC, MVP, MVVM, and MVI for your context |
| `/clean-arch test` | Design testable architecture and apply TDD |
| `/clean-arch diagnose` | "What's wrong with my architecture?" — full diagnostic |

### Team Profiling

Run `/clean-arch team` first. It asks about your team composition, tech stack, experience levels, and product context — information the skills can't derive from code alone. The profile is saved to `.clean-arch-team-profile.md` and automatically loaded by all other skills.

The book is explicit: *"Architectural design is context-dependent."* The same codebase gets very different advice depending on whether it's a 3-person startup or a 50-engineer enterprise team. The team profile captures:

- **People**: Team size, iOS/Android/Flutter/KMP breakdown, seniority, RP knowledge, retention
- **Product**: App type, complexity, offline needs, logic types, data streams per screen
- **Context**: Product stage, expected lifespan, current architecture, biggest pain point

This maps directly to the book's **Technical Price Triangle** (People + Complexity + Longevity) and determines whether you get **Scenario A** (Enterprise) or **Scenario B** (Startup) recommendations.

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

## Usage Examples

### Starting a New Project from Scratch

You're kicking off a new app and want to get the architecture right from day one.

```
You:  /clean-arch team
      → Answer 16 questions about your team, product, and context
      → Profile saved, scenario derived (Startup or Enterprise)

You:  /clean-arch layers
      → "Based on your profile (5 devs, iOS, fintech, 3+ year lifespan),
         I recommend Scenario A (Enterprise) with 4 layers..."
      → Get a concrete layer diagram, model transformation chain,
         and boundary strategy tailored to your context

You:  /clean-arch pattern
      → "Your team knows Combine/async-await, app is complex with 6+
         data streams per screen → MVI-like + Clean Architecture"
      → Get platform-specific implementation guidance (State-model,
         Intent-model, file structure for Swift/Kotlin/Dart)

You:  /clean-arch test
      → "For Enterprise scenario: white-box unit tests per layer,
         Swift Testing framework, protocol-based mocks..."
      → Get testing strategy, pyramid breakdown, and first test examples
```

### Enterprise Project (Large Team, Long-Lived)

Your banking app has 30 engineers across 4 teams. Architecture decisions have lasting impact.

```
You:  /clean-arch team
      → 30 devs, 15 iOS / 15 Android, senior-heavy, RP: yes,
         fintech, 100+ screens, offline, 10+ year lifespan

You:  /clean-arch layers
      → Scenario A: Domain → Application → Presentation & Data → Framework
      → Hard boundaries with feature modules per team
      → Full model chain: ApiModel → DBModel → PlainModel → DomainModel → UiModel
      → Full DIP with DataSource interfaces + DataGateway per Use Case

You:  /clean-arch solid
      → Point it at your codebase
      → "PaymentService violates SCP: it's both an Executor (HTTP calls)
         and Orchestrator (decides cache vs network). Split into
         PaymentRepository (orchestrator) + PaymentApiService (executor)."

You:  /clean-arch review
      → Full architecture review against Clean Architecture principles
      → "Your Room @Entity is used directly in the ViewModel — framework
         data structure crossing the boundary. Create a PlainModel."
```

### Legacy Project (Technical Debt, "We Need to Rewrite")

The app was built fast 3 years ago. Features take forever. Bugs spread everywhere. The team wants to rewrite.

```
You:  /clean-arch team
      → 8 devs, Android, mixed seniority, MVVM (loosely), growth stage,
         biggest pain: "adding features breaks unrelated things"

You:  /clean-arch diagnose
      → Loads team profile, asks for symptoms and key files
      → "Diagnosis: your symptoms (fragility, slow features) map to
         HIGH COUPLING + LOW COHESION — the worst quadrant."
      → Cancer Map: "NetworkManager is a God object (1200 lines,
         handles auth + caching + API calls + error mapping).
         Cancer has spread — 47 classes depend on it directly."
      → Treatment Plan:
         1. Extract AuthService from NetworkManager (highest impact)
         2. Introduce Repository layer between ViewModel and NetworkManager
         3. Define domain models separate from API models
      → "This is NOT a rewrite. It's Preparatory Refactoring — the book
         says refactor right before the feature that needs it."

You:  /clean-arch solid
      → Point it at NetworkManager.kt
      → Gets 5 SCP violations, 2 ISP violations, 3 DIP violations
         with concrete fix steps for each
```

### Solo Developer / Side Project

You're building a personal project on weekends. You want clean code but not enterprise overhead.

```
You:  /clean-arch team
      → Solo, Flutter, simple CRUD, 10 screens, REST API,
         no offline, prototype stage, lifespan uncertain

You:  /clean-arch layers
      → Scenario B (Startup): merged Application + Presentation layers,
         soft boundaries (packages not modules), Repository pattern
      → "For a 10-screen CRUD app, don't create Use Cases — they'd all
         be pass-through middlemen. ViewModel → Repository → Service
         is your MVA. Add layers when complexity demands it."

You:  /clean-arch pattern
      → "Solo dev, simple app, you know RP → MVVM with Riverpod.
         Don't use MVI/BLoC — overkill for this complexity."
```

### Migrating Architecture (e.g., MVC to MVVM, or Adding Clean Architecture)

Your iOS app started with MVC and it's showing its age. You want to modernize incrementally.

```
You:  /clean-arch team
      → 6 iOS devs, mixed seniority, MVC (current), growth stage,
         biggest pain: "Massive View Controllers, can't write tests"

You:  /clean-arch diagnose
      → "Your MVC has untestable controllers (framework code mixed with
         business logic) and no separation between presentation and
         domain. The book says MVC is obsolete — never opt in."
      → Treatment Plan:
         1. Don't rewrite all at once. Apply Preparatory Refactoring.
         2. For the NEXT feature: extract ViewModel from its ViewController
         3. Move business logic to a Use Case, inject via protocol
         4. Write unit tests for the new ViewModel (proves the pattern)
         5. Gradually migrate other screens as you touch them

You:  /clean-arch pattern
      → "Your team is learning async/await → MVVM with @Observable.
         Migrate screen by screen. New screens: MVVM. Old screens:
         migrate when you add a feature to them (Preparatory Refactoring)."

You:  /clean-arch solid
      → Review the first migrated screen
      → Validates the new pattern is correctly applied before you
         repeat it across the codebase
```

### Multi-Platform Team (KMP / Shared Codebase)

Your team shares business logic across iOS and Android with Kotlin Multiplatform.

```
You:  /clean-arch team
      → 12 devs (4 iOS, 4 Android, 4 KMP shared), senior-heavy,
         e-commerce, 50 screens, offline, growth stage

You:  /clean-arch layers
      → "KMP enforces the Dependency Rule architecturally — shared
         module CANNOT depend on platform modules. Your domain and
         application layers live in commonMain (pure Kotlin, zero
         platform deps). Platform code is the Framework layer."
      → Loads platform-kmp.md reference for KMP-specific guidance
      → Model chain: shared ApiModel → shared DomainModel →
         Android UiModel (Compose) / iOS UiModel (SwiftUI)

You:  /clean-arch test
      → "Write tests in commonTest — they run on ALL platforms.
         This is KMP's biggest advantage. 70% of your tests should
         be in commonTest. Platform tests only for platform-specific code."
```

---

## How It Works

The skills don't summarize the book — they **apply** its frameworks to **your** code. When you run `/clean-arch review`, the advisor:

1. Loads your team profile (platform, complexity, team size)
2. Loads the matching platform reference (iOS, Android, Flutter, or KMP)
3. Reads your actual code
4. Classifies each class (Executor / Orchestrator / Data Container)
5. Checks against S.O.L.I.D checklists and the Dependency Rule
6. Assesses MVA fitness (over-engineered? under-engineered? stale?)
7. Delivers prioritized, actionable fixes tailored to your scenario

## Project Structure

```
SKILL.md                              # Main orchestrator + routing
references/
  core-principles.md                   # Book philosophy, mental models, key quotes
  solid-checklist.md                   # All 5 principles with red-flag checklists
  architecture-layers.md               # Clean Architecture layers, 7 logic types, boundaries
  pattern-decision-tree.md             # MV* comparison and decision flow
  testing-strategy.md                  # Testing pyramid, TDD schools, strategies
  platform-ios.md                      # Swift/SwiftUI mapping + community rules
  platform-android.md                  # Kotlin/Compose mapping + community rules
  platform-flutter.md                  # Dart/BLoC/Riverpod mapping + community rules
  platform-kmp.md                      # KMP shared/platform mapping + community rules
skills/
  clean-arch-team/SKILL.md             # Team profiling (start here)
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
