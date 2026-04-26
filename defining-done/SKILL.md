---
name: defining-done
description: Verify a task is truly complete before declaring it done — tests pass, types check, lint passes, SPEC acceptance criteria are met, no debug code or stray TODOs remain, edge cases are handled, behavior matches the spec, and no scope creep was smuggled in. Use before saying "done", before opening a PR, and as the final step in any implementation session. Skip for purely exploratory work or when the user has explicitly said "just sketch it, don't worry about completeness".
---

# Defining Done

The single most common silent failure of AI coding is declaring "done" when the work isn't. The new test passes, so Claude says done — meanwhile the full suite wasn't run, types are broken in adjacent files, the SPEC's edge cases aren't handled, and debug code is still scattered through the diff.

This skill is the verification gate run *by the implementer*, before the work is handed off. It is judgment-level: deterministic checks (lint, typecheck, format) belong in pre-commit hooks; this skill handles the parts that need a thinking pass.

A different skill — `reviewing-as-staff-engineer` — does the same kind of skeptical pass on someone else's diff from fresh context. Use that one for review. Use this one for self-check.

## Process

Run this before declaring any non-trivial implementation complete.

1. **Pull up the canonical definition of done.** Look for `SPEC.md` and `PLAN.md`; if either is present, its acceptance criteria are the contract. If neither exists, derive an explicit acceptance list from the user's original request and any clarifications in this session — and write it down so the walk-through in step 3 has something to check against. Don't substitute "this seems to work" for an explicit list.
2. **Run the full feedback loop, not the partial one.** Tests (all of them, not just the new one), typecheck, lint, and any project-specific checks (from `CONTEXT.md` if present, otherwise discovered from `package.json`/`Makefile`/`pyproject.toml` or asked of the user). If any fail, you are not done.
3. **Walk the acceptance criteria one by one.** For each item — from the SPEC if one exists, otherwise from the list you wrote in step 1 — point at the test or behavior that proves it. If you can't, you are not done.
4. **Check edge cases.** If a SPEC exists, walk its edge cases section. Otherwise, walk the obvious failure modes for this change (empty input, oversized input, partial failure, concurrent access, idempotency — whichever apply). Each should have a test or a documented decision (e.g., "deferred to a follow-up"). If an edge case is silently unimplemented, you are not done.
5. **Diff yourself.** Look at the actual diff — every changed file, every added line. Look for the failure modes listed below.
6. **Resolve or transfer Open questions.** If the SPEC (or your in-session clarifications) had open questions, each one should either be answered and reflected in the implementation, or carried forward into a tracked follow-up. Silently ignoring them is not done.
7. **Produce a Done Statement** (format below). If anything is missing, the statement is "NOT done — here's what's missing." If everything checks out, the statement is "Done — here are the verifications I ran."

## Rules

- **Tests passing is necessary, not sufficient.** Tests pass when the test you wrote matches the code you wrote. They don't verify the acceptance criteria were implemented; they verify your implementation matches your test. The criteria walk-through (against the SPEC if present, otherwise against the explicit list derived from the request) is the check that matters.
- **Run the full suite, not just the file you touched.** AI changes routinely pass their local test while breaking tests elsewhere. The full suite is the only honest check.
- **Refuse to mark done falsely.** If the user says "just call it done, we'll fix it later," push back with specifics: "The acceptance criterion 'handles empty cart' has no test and no implementation. Marking done means we ship that gap. Want me to add a test, or document it as a known follow-up?"
- **Don't conflate "works on the happy path" with "done."** The happy path is the easiest 20% of done. Edge cases, errors, and the behaviors-must-not-change list are most of the work.
- **Prefer hooks for deterministic checks.** If a project doesn't have pre-commit hooks running lint/format/typecheck, recommend setting them up — those checks should be enforced, not just remembered.
- **Treat open questions as required reading when present.** If the SPEC (or the in-session clarifications) listed open questions, none of them should ship silently unresolved. Many AI-implemented features fail here; this skill catches it.
- **Don't add scope while verifying.** If you find a bug while running the done-check, note it and finish the check. Mixing in unrelated fixes during verification means the done-check doesn't end. Bugs found go on the follow-up list.

## Failure modes to look for in the diff

- **Debug code left in.** `console.log`, `print(`, `dbg!`, breakpoints, `if (DEBUG)`. Search the diff explicitly.
- **Commented-out code blocks.** If they're not needed, delete them. If they are, why are they commented?
- **TODOs added in this diff.** A TODO from before is fine; a TODO you just wrote is unfinished work in disguise. Either resolve it or file it.
- **`any` casts (TypeScript), `# type: ignore` (Python), `unwrap()` chains (Rust).** Anywhere the type system was bypassed to make the code compile. Each one is a potential bug.
- **Magic literals.** Numbers or strings that escaped getting names. If the SPEC said "30 days" and the code has a bare `30`, that's a Replace Magic Number with Named Constant waiting to happen.
- **Inconsistent naming with the project's domain vocabulary.** If `UBIQUITOUS_LANGUAGE.md` exists, check it for canonical terms. If not, check that new identifiers match the terminology already used in the surrounding code; new code introducing fresh synonyms re-introduces drift.
- **Files touched outside the planned scope.** If `PLAN.md` named the affected modules, anything you touched outside that list is either a wrong plan or scope creep — flag it. If no plan exists, sanity-check the diff against the original request: are these the files a reasonable implementation would have touched?
- **New dependencies added.** Any new package in `package.json`, `pyproject.toml`, etc. should match the plan. Surprise dependencies have license, security, and maintenance implications.
- **Generated code modified by hand.** Anything in `generated/`, `gen/`, `*.generated.*` should not appear in your diff. If it does, you're patching something that will get overwritten.

## Output format: the Done Statement

Produce one of these two:

### When done

```
## Done Statement

**Task:** <one line — what was implemented>
**Source:** <SPEC.md path if present, PLAN.md path if present, or "user request" with a one-line summary of the explicit acceptance list>

**Verifications run:**
- [x] Full test suite: <N tests pass, 0 fail>
- [x] Typecheck: clean
- [x] Lint: clean
- [x] Format: clean (or: handled by pre-commit hook)
- [x] <project-specific check, e.g., "DB migrations apply cleanly">

**Acceptance criteria walk-through:** (criteria from the SPEC if present, otherwise from the explicit list derived from the request)
- [x] <Criterion 1> → verified by test `<test name>` in `<file>`
- [x] <Criterion 2> → verified by manual test: <what you ran and observed>
- ...

**Edge cases:**
- [x] <Edge case 1> → handled by `<code reference>`, tested by `<test reference>`
- [x] <Edge case 2> → deferred (see Follow-ups)

**Open questions resolved:** (omit if there were none)
- <Q1>: answered as <A>, reflected in <code reference>
- <Q2>: deferred — filed as <ticket/note>

**Diff hygiene:**
- No debug code, no commented-out blocks, no new TODOs, no type-system bypasses.
- Files changed match `PLAN.md` scope (or, if no plan: match the scope a reasonable implementation of the request would touch).
- No new dependencies (or: <dep> added, matches plan/request).

**Follow-ups (not blocking done):**
- <issue or note for things deliberately deferred>
```

### When NOT done

```
## NOT Done — Missing Items

**Task:** <one line>
**Source:** <SPEC.md / PLAN.md / "user request" — whichever was canonical>

**Blocking gaps:**
- <Acceptance criterion X> has no test and no implementation
- Edge case "<empty input>" is not handled — code throws on `[]`
- Typecheck fails in `<file>` — needs fix
- TODO added in `<file:line>` — must resolve or file as follow-up before done

**Recommendation:**
- <smallest path to actually done — usually a short list of concrete actions>

I have not run the Done Statement; the work is not yet complete.
```

## Anti-patterns to refuse

- **"Tests pass, ship it."** Refuse. Tests passing is one of ten checks, not the whole check.
- **"The edge cases are nice-to-haves; the main path works."** Refuse. Acceptance criteria — whether from the SPEC or from the explicit list derived from the request — are the contract. Edge cases listed there are not nice-to-haves; they're requirements with a different name.
- **Marking done with known TODOs in your own diff.** Refuse. Either resolve them or move them to a follow-up list with a real artifact (issue, note, ticket).
- **Skipping the full-suite test run because "I only changed one file."** Refuse. The full suite catches the cross-file breakage that makes AI implementations dangerous.
- **Silently dropping open questions.** Refuse. Whether the questions came from a SPEC or from in-session clarifications, each one needs a documented disposition: answered, deferred-with-ticket, or escalated to the user.

## Integration with hooks

For the deterministic parts of "done" — lint, format, typecheck, secret scan — pre-commit hooks are the right enforcement mechanism. They run every time and don't depend on memory. If the project doesn't have them, point at `setup-pre-commit` (Pocock's skill, or the project's existing tooling) and recommend it.

This skill handles the *judgment* parts that hooks can't:
- Did you implement what was actually asked for (per the SPEC if present, or per the explicit acceptance list derived from the request)?
- Did you handle the edge cases?
- Did you resolve the open questions?
- Did the diff stay within scope?
- Is your output free of self-introduced TODOs and debug code?

Hooks enforce; this skill verifies. Both are needed.

## Post-output instruction

After producing the Done Statement, state one of:

> **Done.** Open a PR or hand off as appropriate. Recommend running `reviewing-as-staff-engineer` from a fresh session for an external check before merging.

or

> **Not done.** The blocking gaps above need to be closed first. Want me to address them now, or document them as deferred and produce an honest "partially complete" handoff?
