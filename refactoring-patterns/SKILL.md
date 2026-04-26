---
name: refactoring-patterns
description: Execute refactorings as named transformations from Fowler's catalog (Extract Function, Inline Variable, Move Method, Replace Conditional with Polymorphism, etc.) and Ousterhout's complexity moves (deepen module, hide information, pull complexity downwards). Tests must stay green between every transformation. Use when cleaning up AI-generated verbosity, removing duplication, simplifying interfaces, or after a code review reveals smells. Skip for changes that alter observable behavior — those are not refactorings.
---

# Refactoring Patterns

Refactoring is a structural change that does not alter observable behavior. Every transformation has a name. Tests stay green between every transformation. If tests aren't green, you stopped refactoring and started rewriting.

AI's natural failure mode is to "improve" code by rewriting large sections at once, which silently changes behavior, breaks tests, or both. This skill forces named, mechanical, test-guarded steps instead.

The skill does not find smells — that's `improve-codebase-architecture` or a code review. It executes the fix.

## Process

For each smell or improvement:

1. **Name the transformation before doing it.** "I'm going to Extract Function on lines 42–58 into `validatePayload`." If you can't name it, you don't have a refactoring — you have a rewrite. Stop. For a rewrite that crosses module boundaries, `planning-by-modules` is the right next step; for a smaller behavior change, plan it as a deliberate change with its own test rather than disguising it as cleanup.
2. **Run tests. Confirm green.** If tests are red before you start, refactoring is not safe. Make them green first, or run `characterizing-legacy` to add coverage before refactoring.
3. **Apply the transformation.** Make the smallest mechanical change that achieves the named refactoring.
4. **Run tests immediately.** All of them, not just the file you touched. Run typecheck and lint too.
5. **If anything fails, revert.** The transformation was unsafe — there was a hidden behavior change, or tests were missing coverage. Don't try to fix forward.
6. **Commit (or note the checkpoint).** One refactoring per commit makes review and revert easy. If you're not committing, mention the named transformation in the running summary so the user can track.
7. **Move to the next transformation.** Don't batch. The whole point is the test-green checkpoint between each.

## Rules

- **Behavior is sacred.** A refactoring that breaks a test changed behavior, by definition. Either the refactoring was wrong, or the test was wrong — investigate before continuing.
- **Name the move.** Use Fowler's catalog vocabulary (Extract Function, Inline Variable, Move Method, Replace Magic Number with Named Constant, Replace Conditional with Polymorphism, Introduce Parameter Object, Decompose Conditional, Slide Statements, etc.). Naming forces the move to be a single, mechanical thing.
- **One transformation per step.** "Extract Function and rename it and add a parameter" is three transformations. Do them one at a time, tests green between each.
- **Don't refactor on a red branch.** If tests are failing for any reason, refactoring is not safe.
- **Don't change behavior under cover of refactoring.** If you find a bug while refactoring, stop. Either fix the bug separately (with a test), or finish the refactoring first and then fix the bug.
- **Prefer small, common transformations over large architectural moves.** Extract Function dozens of times will reshape a file more reliably than one big restructure.
- **Run the full feedback loop after each step**, not at the end. Tests, typecheck, lint. Same loop as `tdd-with-feedback-loops`.

## The transformations you'll use most

A handful of Fowler's transformations cover the vast majority of cleanup. Know these by name:

- **Extract Function** — pull a fragment of code into its own named function. The most useful refactoring; reach for it whenever you can name what a fragment does.
- **Inline Function** — opposite. When a function's body is as clear as its name, inline it.
- **Extract Variable** — give a sub-expression a name. Useful before further extraction.
- **Inline Variable** — when a variable adds no clarity beyond the expression it holds.
- **Rename Variable / Rename Function** — when the name lies, mislead, or follows an old vocabulary. Often runs after `ubiquitous-language` updates.
- **Replace Magic Number with Named Constant** — every literal that has meaning gets a name.
- **Move Function** — when a function lives on the wrong module. Often paired with Ousterhout's "pull complexity downwards."
- **Introduce Parameter Object** — when several parameters always travel together.
- **Decompose Conditional** — extract the condition, the then-branch, and the else-branch into named pieces.
- **Replace Conditional with Polymorphism** — when a switch/if-chain dispatches on type or kind, replace with a polymorphic structure. Powerful but heavy — use only when the conditional appears in multiple places.
- **Combine Functions into Class** — when several functions share a prefix and operate on the same data, group them.
- **Split Phase** — when one function does two things in sequence (parse then process, validate then transform), separate them.

For anything beyond these, look up the name in Fowler's *Refactoring* (2nd ed.) or describe what you're doing in plain language and verify it's a named transformation.

## Ousterhout's complexity moves

These are higher-level than Fowler's transformations and often map onto sequences of them:

- **Deepen a module.** Reduce the public interface, move logic inward. Often a series of Move Function and Inline Function transformations toward a single public entry point.
- **Hide information.** Stop exposing a field or implementation detail through the interface. Often Encapsulate Variable then Move Function.
- **Pull complexity downwards.** Make the caller's life simpler at the cost of more logic inside the module. Often Inline (at the callsite) then Extract (inside the module).
- **Eliminate a pass-through method.** Delete a method that does nothing but call another with the same signature. Inline Function on every callsite.
- **Define errors out of existence.** Change an interface so the error case can't occur. Heavier than a refactoring — often crosses into design change. If it does, stop and treat it as a planned design change (running `planning-by-modules` is one good option) rather than as a behavior-preserving refactor.

## Example: Extract Function, step by step

Before:
```typescript
function processOrder(order: Order) {
  // ... 30 lines of code ...
  if (order.items.length === 0) throw new Error("empty");
  for (const item of order.items) {
    if (item.quantity < 1) throw new Error("invalid quantity");
    if (!item.sku) throw new Error("missing sku");
  }
  // ... more code ...
}
```

Named transformation: Extract Function on the validation block.

After:
```typescript
function processOrder(order: Order) {
  // ... 30 lines of code ...
  validateOrderItems(order);
  // ... more code ...
}

function validateOrderItems(order: Order) {
  if (order.items.length === 0) throw new Error("empty");
  for (const item of order.items) {
    if (item.quantity < 1) throw new Error("invalid quantity");
    if (!item.sku) throw new Error("missing sku");
  }
}
```

Test the change: every test that exercises `processOrder` continues to pass. No test changes. That is what behavior preservation looks like.

## Anti-patterns to refuse

- **"Improve this file" without a named transformation.** Refuse. Ask what specifically to improve, or run `improve-codebase-architecture` to find candidates first.
- **Batched refactorings without test runs between.** Refuse. The point of refactoring discipline is the green checkpoint between steps.
- **Refactoring without tests.** If the code has no tests, you cannot verify behavior preservation. Run `characterizing-legacy` first.
- **"Just clean it up while you're there."** Mixing a behavior change with a refactoring in the same commit means you can no longer tell which broke the test. Separate them.
- **Renaming away from the project's domain vocabulary.** If `UBIQUITOUS_LANGUAGE.md` exists, refactoring renames must align with it (update the glossary first if they shouldn't). If no glossary exists, align with the terminology already established in the surrounding code.

## Post-output instruction

After completing refactorings, summarize:

> Applied N named refactorings: <list, one per line, with the named transformation>. All tests, typecheck, and lint pass after each step. No behavior changes.
