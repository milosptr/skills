# Agent Skills

A collection of skills for Claude Code that turns AI from a fast code generator into a disciplined engineering partner. Each skill encodes one piece of strategic control: clarification before code, planning by module boundaries, test-first feedback loops, refactoring with a safety net, named-knowledge duplication checks, and review from fresh context.

The set is built on the thesis that **AI does not make software fundamentals obsolete — it makes them more important**. Generating code is cheap; owning bad code is expensive. These skills are the discipline that keeps AI-assisted code maintainable.

The skills compose. SPEC → CONTEXT → PLAN → implementation → review, each step in a fresh session, with file artifacts as the bridges. Run them in order for a full feature; run individually for targeted work.

## Planning & Design

These skills help reach shared understanding and pick the right shape before any code is written.

### clarifying-intent
> Interview the user about a feature, refactor, or change until you reach shared understanding, then write a SPEC.md they confirm. Use before implementing anything that crosses more than one file or has unstated assumptions.

Inspired by Frederick P. Brooks's "design concept" from *The Design of Design* and Matt Pocock's `grill-me` pattern.

### mapping-codebase
> Produce a curated CONTEXT.md map of an unfamiliar or large codebase covering entry points, architecture, modules, domain terms, commands, and landmines.

Built on Boris Cherny and Shrivu Shankar's published Claude Code workflows: context degradation is the primary failure mode, and a curated map is the artifact that fixes it.

### ubiquitous-language
> Extract a DDD-style domain glossary from the codebase and conversation, flag ambiguities and synonyms, propose canonical terms, save to UBIQUITOUS_LANGUAGE.md.

From Eric Evans's *Domain-Driven Design* and the "project glossary" tip in *The Pragmatic Programmer*. Pocock has a similar skill of the same name; this one extends the pattern to scan the codebase as the source of truth and includes a code-identifier column.

### designing-it-twice
> Generate two or three radically different interface designs for a module, each under a different design constraint, then compare and let the user pick.

From John Ousterhout's *A Philosophy of Software Design*, chapter 11. The first plausible interface is always plausible — that is not a signal it is right.

### planning-by-modules
> Convert a SPEC, PRD, or feature request into a module-level implementation plan that names affected modules, designs their public interfaces, lists internal changes, identifies dependencies and risks, and specifies tests at each boundary.

Combines Ousterhout's deep-modules-and-information-hiding principles with the "plan by modules, not files" rule from Matt Pocock's *Software Fundamentals Matter More Than Ever* talk.

## Development

These skills produce, refactor, and safety-net code.

### tdd-with-feedback-loops
> Implement features or fixes using test-first development with strict vertical slicing — one failing test, smallest passing change, then typecheck and lint and tests as the full feedback loop, then refactor.

Extends Kent Beck's TDD discipline and Matt Pocock's `tdd` skill with the broader "rate of feedback is the speed limit" framing — typecheck and lint, not just tests, run between every cycle.

### characterizing-legacy
> Add characterization tests around untested code before modifying it, capturing what the code actually does (not what it should do) as a safety net for refactoring or change.

From Michael Feathers's *Working Effectively with Legacy Code*. The counterintuitive core: write a test, run it, paste the actual output as the expected value. Document behavior, not desired behavior.

### refactoring-patterns
> Execute refactorings as named transformations from Fowler's catalog and Ousterhout's complexity moves. Tests must stay green between every transformation.

From Martin Fowler's *Refactoring* (2nd edition) and John Ousterhout's *A Philosophy of Software Design*. Behavior preservation is the test of a refactoring; without it, you are rewriting.

### enforcing-dry
> Find and evaluate duplication in code, distinguishing true knowledge duplication (must be unified) from coincidental similarity (must be left alone).

From *The Pragmatic Programmer* and Martin Fowler's Rule of Three. DRY is about knowledge, not syntax — the most damaging AI failure mode is extracting on syntactic similarity, producing fake abstractions over unrelated code.

## Verification & Review

These skills close the loop before declaring work shippable.

### defining-done
> Verify a task is truly complete before declaring it done — tests pass, types check, lint passes, SPEC acceptance criteria are met, no debug code or stray TODOs remain, edge cases are handled, behavior matches the spec, and no scope creep was smuggled in.

From Boris Cherny's "would a staff engineer approve this?" rule, run by the implementer as a self-check before opening a PR. Premature completion claims are the most common silent AI failure; this skill is the gate that catches them.

### reviewing-as-staff-engineer
> Review a diff or PR from a fresh-context perspective, challenging implementation shortcuts, hidden assumptions, missing tests, security risks, design regressions, and maintainability decay.

The external check, run from a fresh session by a Claude that did not write the code being reviewed. Pairs with `defining-done` (self-check by implementer) — together they catch what neither alone would.

## Installation

Each skill is a folder containing a single `SKILL.md`. Place them under `~/.claude/skills/` for personal use, or `.claude/skills/` for project-only.

```
~/.claude/skills/
├── clarifying-intent/SKILL.md
├── mapping-codebase/SKILL.md
├── ubiquitous-language/SKILL.md
├── designing-it-twice/SKILL.md
├── planning-by-modules/SKILL.md
├── tdd-with-feedback-loops/SKILL.md
├── characterizing-legacy/SKILL.md
├── refactoring-patterns/SKILL.md
├── enforcing-dry/SKILL.md
├── defining-done/SKILL.md
└── reviewing-as-staff-engineer/SKILL.md
```

Claude Code watches these directories and picks up new or edited skills within the current session.

## Sources

- John Ousterhout, *A Philosophy of Software Design* (2018)
- Andrew Hunt and David Thomas, *The Pragmatic Programmer* (1999, 20th anniversary ed. 2019)
- Martin Fowler, *Refactoring: Improving the Design of Existing Code* (2nd ed., 2018)
- Michael Feathers, *Working Effectively with Legacy Code* (2004)
- Eric Evans, *Domain-Driven Design* (2003)
- Frederick P. Brooks, *The Design of Design* (2010)
- Matt Pocock, *Software Fundamentals Matter More Than Ever* (talk) and the [mattpocock/skills](https://github.com/mattpocock/skills) repository
- Boris Cherny's published Claude Code workflow and CLAUDE.md configuration
- Anthropic's [Agent Skills documentation](https://docs.claude.com) and skill authoring best practices
