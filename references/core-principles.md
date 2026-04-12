# Clean Mobile Architecture -- Core Principles

**Source**: "Clean Mobile Architecture" by Petros Efthymiou (2022)

## The Golden Circle

The book follows Simon Sinek's "Start with Why":
1. **WHY** -- Maximize effectiveness and efficiency in building software
2. **HOW** -- Fundamental Software Objectives, S.O.L.I.D, Separation of Concerns
3. **WHAT** -- Actionable architectural patterns for mobile apps

## Fundamental Software Objectives (FSO)

Code must be: **Readable, Testable, Extensible, Robust, Maintainable, Resilient & Durable**

The bridge from business objectives to technical implementation is **SCC**:
- **S**eparation of Concerns
- **(High) C**ohesion
- **(Low) C**oupling

## Key Mental Models

### The Loan Metaphor
Technical debt = financial loan with interest. Quick and dirty = high-interest loan. If not prepaid, leads to "house foreclosure" (project disposal/rewrite).

### Contain Cancer
Not all bad decisions are equally dangerous. Isolate legacy/third-party code behind boundaries. Contained cancer can be treated later; spread cancer is fatal.

### Products vs. Projects
Software is a product (evolving, open-ended), not a project (fixed scope/time). "Software must be soft -- easy to change and reshape." (Robert C. Martin)

### Technical Price Triangle
Architecture elaborateness depends on: **People** (team size, seniority, retention) + **Feature Complexity** (types of logic, dependencies) + **Longevity** (how long it will live).

### Minimum Viable Architecture (MVA)
"The simplest architecture that can support the clean implementation of the product."
- **Minimum**: No speculation, no unnecessary boundaries. Avoid segregating into feature modules from the start unless needed.
- **Viable**: Respects S.O.L.I.D, Clean Architecture, and best practices.
- MVA evolves alongside system complexity via **Preparatory Refactoring** (refactor right before a feature the codebase can't absorb).

### The Mastery Path (Dunning-Kruger)
- **Junior**: Struggles with patterns, avoids them
- **Mid-level**: Understands patterns, overuses them -- "complexifies simple code"
- **Senior**: Knows when each tool is helpful and when it is not

**"Design patterns are meant to simplify complex code, not complexify simple code."**

## Decision Framework

"Whenever you find yourself adding a pattern, ask: 'Which high-level objective does it serve and how?' If you can't answer, abandon it and write simpler code."

## Key Quotes

1. "Writing cheap code ends up being expensive."
2. "The only way to go fast is to go well." -- Robert C. Martin
3. "If you think good architecture is expensive, try bad architecture." -- Brian Foote
4. "Architecture is a hypothesis that needs to be proven by implementation and measurement." -- Tom Gilb
5. "Always implement things when you actually need them, never when you just foresee that you need them." -- Ron Jeffries (YAGNI)
6. "Make the change easy, then make the easy change." -- Kent Beck
7. "Science is about knowing; engineering is about doing." -- Henry Petroski
