---
name: surveying-architecture
description: Survey existing code for architectural problems and propose specific, named improvement opportunities — shallow modules to deepen, information leakage to hide, pass-through methods to eliminate, temporal decomposition to reorganize, knowledge duplication to unify. Output is a severity-ranked list of findings, each tied to a named transformation. Use when arriving at an unfamiliar or messy codebase and wanting to know what to fix, when a refactoring effort needs a target list, or when planning architectural cleanup. Skip for cosmetic preferences, style debates, or whole-codebase audits — work in scoped areas (a module, a directory, a tangle the user names).
---

# Surveying Architecture

This is the *finder* skill. It reads existing code, identifies architectural problems, and proposes specific transformations to fix them. It does not execute the fixes — that's `refactoring-patterns`. It does not produce a neutral map — that's `mapping-codebase`. It does not review a diff — that's `reviewing-as-staff-engineer`.

The bias to fight is **false-positive aesthetic complaints**. Every codebase has things that "could be cleaner." A survey that lists everything is worse than no survey, because it trains the user to ignore the output. A survey that names *real architectural problems* — shallow modules, information leakage, knowledge duplication — and proposes *named transformations* to fix them is what's useful.

If a finding can't be named in Ousterhout's vocabulary or fixed by a named refactoring, it's a nit, not a finding. Refuse it.

## Process

1. **Scope the survey.** Ask the user what to look at: a module, a directory, a feature area, or a specific tangle they've named. Do not survey the whole codebase. Whole-codebase surveys produce 100-item lists nobody acts on.
2. **Read the targeted code.** Read the public interfaces of the modules in scope and the top-level structure of their implementations. Don't read every line — read enough to spot the patterns below.
3. **Walk the lenses (see below) over the scoped code.** For each lens, look for instances. Cite file:line. Don't speculate — show the actual code.
4. **Apply the deletion test to anything you suspect is shallow.** Would deleting this module concentrate complexity (it was deep — leave alone) or just move it elsewhere (it was shallow — candidate to deepen)? The "concentrates" answer is the signal.
5. **Name the transformation.** Each finding must propose a specific named refactoring (Extract Function, Inline Function, Move Function, Replace Conditional with Polymorphism, Combine Functions into Class, etc.) or a specific Ousterhout move (deepen module, hide information, pull complexity downwards, eliminate pass-through, define errors out of existence). If you can't name the fix, the finding isn't real — drop it.
6. **Tag severity.** Each finding is **Critical**, **Worth doing**, or **Nit**. Distribute honestly — most findings will be Worth doing; a survey where everything is Critical is fear-mongering, and one where everything is Nit isn't worth running.
7. **Cap the output.** No more than ~10 findings per survey, ranked. If you have more, you're either over-surveying or not ranking. The list of "everything that could be better" is not a useful artifact.
8. **Output the survey** in the format below. End with explicit handoffs to executor skills.

## The lenses

Walk these in order. For each lens, look for specific instances and tag severity.

### 1. Shallow modules (Ousterhout)

A module is shallow when its interface costs almost as much to use as its implementation does to write. Classic shape: one function wrapping one library call with no added abstraction. Symptom: callers must understand both the module's interface *and* the underlying mechanism it delegates to. The module is a tax, not a hider.

**Look for** modules with public interfaces nearly as wide as their implementations, one-line wrapper functions, classes whose every method delegates one-to-one. **Apply the deletion test:** would removing this module force callers to do meaningfully more work, or merely the same work at a different address? "Same work, different address" — shallow.

**Fix:** deepen the module (Inline Function on shallow wrappers, then Extract Function around the actual logic in the new location with a wider responsibility), or eliminate the module entirely if the wrap was vacuous.

### 2. Information leakage (Ousterhout)

A design decision is leaked when callers must know it to use the module correctly. Symptoms: the same concept (a date format, a database column convention, a protocol detail) repeated in many places; changing one of them requires changing all of them; the module's "secret" isn't actually secret.

**Look for** literals or constants repeated across modules that all encode the same fact, the same parsing/formatting at multiple call sites, structural assumptions about another module's internals. **Fix:** Encapsulate Variable, then Move Function into the module that owns the secret.

### 3. Pass-through methods (Ousterhout)

A method that does nothing but call another method with the same signature. Symptoms: one-line bodies that just delegate; matching signatures across two layers; a layer of abstraction whose only purpose is to be a layer.

**Look for** methods whose body is `return other.sameMethod(args)`, classes where every method is a pass-through to a held collaborator. **Fix:** Inline Function on every pass-through call site, eliminating the layer.

### 4. Temporal decomposition (Ousterhout)

Modules organized by *when* operations happen rather than *what knowledge they own*. Symptoms: classes named `Step1Processor`, `Phase2Handler`, `PreSaveValidator`, `PostSaveNotifier`; modules whose responsibility is "things that run before X."

**Look for** module names that describe sequence position rather than domain knowledge, modules that exist only because something runs before/after them. **Fix:** reorganize by knowledge — Combine Functions into Class for modules that own the same domain concept across phases, or pull lifecycle ordering out of module structure into an explicit pipeline.

### 5. Conjoined methods / temporal coupling

Two methods that must be called in a specific order, with no enforcement. Symptoms: `init()` then `process()`; documented preconditions instead of structural ones.

**Look for** methods whose docstrings say "must be called after X", APIs where misuse is easy and silent. **Fix:** restructure so misuse is impossible — return a different object from the first call that exposes the second, collapse the two into one, or use the type system to encode order.

### 6. Knowledge duplication (Pragmatic Programmer)

The same fact about the world represented in multiple places. Distinct from coincidental syntactic similarity (see `enforcing-dry`).

**Look for** business rules expressed in two modules, tax/validation/formatting rules duplicated across callsites. **Fix:** hand off to `enforcing-dry` for the knowledge-vs-coincidence judgment, then `refactoring-patterns` for execution.

### 7. Generic-by-design failure (Ousterhout)

A module that tried to be general-purpose and ended up serving none of its callers well. Symptoms: many configuration parameters, long parameter lists with mostly-default values, callers all using the same combination of options to get the behavior they actually wanted.

**Look for** modules with `options` objects whose every caller sets the same fields, APIs with many flags no caller exercises in combination. **Fix:** pull complexity downwards — make the common case trivial, allow escape hatches for the rare. Often Inline Function (at the callsite) then Extract Function (inside, with the right shape).

### 8. Hard-to-describe modules

If you cannot describe a module's responsibility in one sentence without using "and," it probably has more than one. Symptoms: names like `Utils`, `Helper`, `Manager`; responsibilities that read as a list.

**Look for** modules whose name is generic, modules whose public interface mixes unrelated concerns. **Fix:** split by single knowledge — Combine Functions into Class with one responsibility per class.

## Rules

- **Each finding must name the architectural pattern AND the named transformation.** "This is messy" is not a finding. "Module `formatHelpers` is shallow (4 one-line wrappers around `Intl.NumberFormat`); deepen by Inline Function on the wrappers and reorganize remaining logic into `pricing/format.ts` with a named `formatPrice` interface" is a finding.
- **Refuse aesthetic findings.** "I'd organize this differently" is not a finding. "This file is long" is not a finding. "I prefer X pattern over Y pattern" is not a finding. Each finding must point to a *named architectural problem* with a *named fix*.
- **Cite file:line for every finding.** Vague surveys are useless. Show the actual code.
- **Apply the deletion test before claiming a module is shallow.** Without that check, you're guessing.
- **Cap at ~10 findings.** If you have more, rank harder. Surveys are useful to the extent they're acted on; long lists aren't.
- **Severity must be honestly distributed.** A survey where everything is Critical is panic. A survey where everything is Nit isn't worth running. Most findings will be Worth doing.
- **Don't propose architecture changes that contradict an explicit decision.** If `CONTEXT.md` or an ADR says "we use the repository pattern here even though it's shallow because of testing constraints," don't list "shallow repositories" as findings. If you're not sure whether such a constraint exists, ask.
- **Don't survey the whole codebase by default.** Scope to a module, directory, or area the user names. Whole-codebase surveys produce noise; focused surveys produce action.
- **Don't execute the fixes in this skill.** Hand off. The discipline of survey-then-execute (separate sessions if helpful) keeps the finder honest — it's not protecting its own list of fixes from criticism, it's proposing to a future session.

## Output format

```
# Architecture Survey: <scope — module name, directory, or named area>

**Scope reviewed:** <files/modules read>
**Constraints noted:** <any from CONTEXT.md/ADRs that bound the survey, or "none">

## Findings

### Finding 1: <one-line title>

**Severity:** Critical / Worth doing / Nit
**Pattern:** <named pattern from the lenses — Shallow Module, Information Leakage, Pass-Through, etc.>
**Locations:** `path/to/file.ts:42`, `path/to/file.ts:108`

**Observation:**
<2–3 sentences describing what's there. Show the actual code shape, not an abstraction of it.>

**Why this is a real problem (not aesthetic):**
<one sentence: what cost it imposes on callers, future changes, or testability>

**Proposed transformation:**
<named refactoring(s) and/or Ousterhout move. Be specific: which functions, which direction, what name the new module/function would have.>

**Estimated scope:** <small / medium / large — based on number of callsites or files touched>

### Finding 2: ...

(repeat for up to ~10 findings, ranked by severity then estimated impact)

## Findings deliberately not included

<Optional. Use to mention things you considered and rejected — patterns that look like findings but aren't, given the deletion test or constraints.>
- <Module X looked shallow, but the deletion test showed deletion would scatter complexity across 6 callers — leave deep.>
- <Pattern Y appears 3x but the contexts are different knowledge — coincidental, not duplication.>

## Suggested next steps

1. <For Critical findings: address before further feature work. Run `refactoring-patterns` with the named transformation in a fresh session.>
2. <For Worth-doing findings: schedule as cleanup. Group by file when possible to minimize churn.>
3. <For Nit findings: optional, address opportunistically.>
4. <For knowledge duplication findings: hand off to `enforcing-dry` for the extract-or-leave judgment before executing.>
```

## Anti-patterns to refuse

- **"Survey the whole codebase."** Refuse. Ask the user to scope. Whole-codebase surveys produce noise.
- **Findings without a named transformation.** Refuse. If you can't name how to fix it, the finding isn't real.
- **Findings without file:line citations.** Refuse. Vague findings can't be acted on.
- **Aesthetic findings dressed as architectural ones.** Refuse. "I'd structure this differently" or "this could be cleaner" is not a finding. Each finding must map to a named pattern from the lenses above.
- **Surveys with everything tagged Critical.** Refuse. The reviewer is supposed to rank, not catastrophize.
- **Surveys longer than ~10 findings.** Refuse. Rank harder. The cap forces honesty about priority.
- **Executing the fix during the survey.** Refuse. This skill produces a list. `refactoring-patterns` and `enforcing-dry` execute. Mixing them defeats the survey-then-execute discipline that keeps the finder honest.
- **Reviewing your own past surveys.** If a previous session in this conversation produced a survey, this session should not re-survey the same scope and "find different things." Either trust the previous survey or have a fresh session redo it.

## Integration with other skills

- **`mapping-codebase`** is worth running first if the codebase is unfamiliar — you can't survey what you don't understand. The map gives you the module boundaries; this skill judges them.
- **`enforcing-dry`** is the specialist for the knowledge-duplication lens. If duplication is the dominant finding, hand off to it for the per-instance extract-or-leave judgment.
- **`refactoring-patterns`** executes the named transformations this skill proposes. Run it in a fresh session per finding (or grouped by file) to keep each refactor a clean test-green cycle.
- **`planning-by-modules`** is the right next step if a finding requires changing a module's *public interface* (deepening, splitting, merging) — that's a design change, not just a refactor, and deserves a plan.
- **`designing-it-twice`** applies if a deepening or splitting finding requires picking between multiple plausible new interface shapes.
- **`reviewing-as-staff-engineer`** is the diff-level analog of this skill. They share the Ousterhout lens; the difference is one looks at existing code as-is, the other looks at proposed changes.

## Post-output instruction

After producing the survey, state:

> Identified N findings: <X> Critical, <Y> Worth doing, <Z> Nits. Recommend handing off Critical findings to `refactoring-patterns` (or `planning-by-modules` if interfaces change) in fresh sessions. The survey is the proposal; execution is a separate discipline.
