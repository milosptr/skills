---
name: characterizing-legacy
description: Add characterization tests around untested code before modifying it, capturing what the code actually does (not what it should do) as a safety net for refactoring or change. Use before any refactor of untested code, before changing a function in a tangled module, when AI is about to "improve" code without coverage, or when extending legacy behavior. Skip for code that is already well-covered, or for greenfield code (use tdd-with-feedback-loops instead).
---

# Characterizing Legacy

Legacy code is code without tests. You cannot safely change what you cannot test. Characterization tests are how you put the safety net in place before any change.

The core move is counterintuitive: write a test, run it, see what the code *actually* returns, then make that the expected value. You are documenting current behavior, not asserting correctness. If the current behavior is a bug, the test still goes in — and the bug gets a separate ticket.

This is the prerequisite for `refactoring-patterns` whenever the target code has no coverage. Without characterization, "refactoring" untested code is rewriting in disguise.

## Process

For each function or area you need to safety-net:

1. **Identify the change point.** What specific function, method, or area are you about to touch? Don't characterize the whole codebase — characterize the seam around the change.
2. **Find a test point.** Where can you call this code from a test? If it's not callable from a test harness yet, find a *seam* (see below) and break the dependency that's blocking testability.
3. **Write a test that calls the code with concrete inputs**, asserting on something you *know is wrong* — typically a sentinel value like `assertEquals(actualResult, "WRONG")`. The test must fail.
4. **Run the test.** Read the actual output from the failure message.
5. **Paste the actual output back as the expected value.** This is the characterization. The test now documents what the code does today.
6. **Repeat with edge inputs.** Empty input, oversized input, boundary values, the kinds of inputs that reveal branches. Each one is a new characterization test. If a behavior surprises you, write it down — don't fix it.
7. **When coverage is enough for the change you're about to make**, stop. You don't need to characterize everything — you need to characterize the surface area you're about to touch.
8. **Now refactor or change** with the safety net in place. Use `refactoring-patterns`.

## Rules

- **Document what the code does, not what it should do.** This is the rule the AI most often breaks. If you write what you *think* the function should return, your test will pass against the current bug, pin the bug in place, and give false confidence.
- **Counterintuitive behaviors stay in the test.** If `formatPrice(0)` returns `"$-0.00"`, write `expect(formatPrice(0)).toBe("$-0.00")`. Don't fix the negative-zero bug while characterizing — that's a separate change.
- **Bugs found while characterizing get a ticket, not a fix.** Note them in your summary. Fix them as a separate, deliberate change with the characterization tests still in place to confirm the fix.
- **Use sentinel-then-replace.** Start every test with a deliberately wrong expectation, run, paste actual. Resist the urge to predict the answer. Predictions are how you accidentally write what-it-should-do tests.
- **Don't refactor while characterizing.** Refactoring without tests is the problem you are solving. Adding tests *is* the work — don't try to do both at once.
- **Don't aim for total coverage.** Characterize the change surface. A 200-line legacy function being changed in lines 80–120 needs characterization around that range and its callers, not exhaustive coverage of every branch.
- **Snapshot/golden-master tests are fine for complex outputs.** When the output is a large string, JSON, or generated file, an approval-test snapshot is often the only practical characterization. Use it.
- **Once the test passes, the code is "characterized." Treat it as the contract** until and unless the user explicitly approves a behavior change.

## Seams: how to make untestable code testable

A seam is a place where you can change behavior without editing the code under test. Feathers' canonical categories:

- **Sensing seam** — a place where you can observe what the code does. Adding a logger, capturing a return value, exposing a getter. Use only when the code is otherwise too opaque to test.
- **Separation seam** — a place where you can substitute a dependency. The most common AI move here is **parameter injection**: take a function that calls `Date.now()` or `fetch()` directly, and pass the dependency in as a parameter. Now the test can pass a fake.
- **Object seam** — replace an entire object dependency at construction time. If a class instantiates its own collaborators, change it to accept them via constructor.
- **Link seam / build seam** — replace at link or build time (more relevant for compiled languages with build-system substitution).

If the code is genuinely untestable without a structural change, do the smallest seam-introducing change first. That structural change is itself untested — keep it minimal and obvious enough to review by eye. Then write characterization tests against the new seam.

## Example: characterizing a tangled function

```typescript
// Untested legacy function we need to change
function calculateRefund(orderId: string): number {
  const order = db.findOrder(orderId);  // direct DB dependency
  const now = Date.now();                // direct time dependency
  const daysSince = (now - order.purchaseDate) / 86400000;
  if (daysSince > 30) return 0;
  if (order.status === "shipped") return order.total * 0.5;
  return order.total;
}
```

Step 1 — introduce seams (separation):

```typescript
function calculateRefund(
  order: Order,            // injected, not fetched
  now: number = Date.now() // injected with default
): number {
  const daysSince = (now - order.purchaseDate) / 86400000;
  if (daysSince > 30) return 0;
  if (order.status === "shipped") return order.total * 0.5;
  return order.total;
}
```

Step 2 — write a characterization test with a deliberately wrong expectation:

```typescript
test("characterize: shipped order, 10 days old, total $100", () => {
  const order = { purchaseDate: 0, status: "shipped", total: 100 };
  const now = 10 * 86400000;
  expect(calculateRefund(order, now)).toBe("WRONG");  // sentinel
});
```

Step 3 — run, see actual:

```
Expected: "WRONG"
Received: 50
```

Step 4 — paste actual:

```typescript
test("characterize: shipped order, 10 days old, total $100", () => {
  const order = { purchaseDate: 0, status: "shipped", total: 100 };
  const now = 10 * 86400000;
  expect(calculateRefund(order, now)).toBe(50);
});
```

Step 5 — repeat with edge inputs:

```typescript
test("characterize: order older than 30 days returns 0", () => {
  const order = { purchaseDate: 0, status: "shipped", total: 100 };
  const now = 31 * 86400000;
  expect(calculateRefund(order, now)).toBe(0);
});

test("characterize: shipped at exactly 30 days", () => {
  const order = { purchaseDate: 0, status: "shipped", total: 100 };
  const now = 30 * 86400000;
  expect(calculateRefund(order, now)).toBe("WRONG");  // sentinel — find the boundary
});
// Result: 50. Boundary is exclusive at 30 days. Document that.
```

Now the function is safety-netted. You can refactor or change it with confidence.

## Anti-patterns to refuse

- **Writing tests with predicted outputs instead of running to capture.** Refuse. Use the sentinel-then-paste pattern even if the answer seems obvious. Obvious predictions are wrong more often than you expect.
- **"Fixing" bugs found while characterizing.** Refuse. Note the bug, leave the test reflecting current behavior, file a follow-up. Mixing fixes with characterization defeats the safety net.
- **Aiming for 100% coverage on legacy code.** Refuse. Characterize the change surface. Total coverage is a separate, larger initiative.
- **Skipping characterization because "the change is small."** Even a one-line change in untested code needs a test around it before and after, or you have no way to know if the change broke anything.
- **Asserting on internal state to "make characterization easier."** Refuse. Characterize through the public interface. If that's impossible, find a sensing seam first — but prefer behavior-level characterization.

## Post-output instruction

After characterizing, summarize:

> Added N characterization tests around `<function/area>`. They document current behavior (not desired behavior). Surprising behaviors found: <list, or "none">. The code is now safety-netted for the planned change. Next step: run `refactoring-patterns` or implement the change with `tdd-with-feedback-loops`.
