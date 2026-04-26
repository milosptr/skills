---
name: clarifying-intent
description: Interview the user about a feature, refactor, or change until you reach shared understanding, then write a SPEC.md they confirm. Use before implementing anything that crosses more than one file or has unstated assumptions. Skip for typos, single-line fixes, or tasks where the user already provided a complete spec.
---

# Clarifying Intent

The most expensive AI failure is building the wrong thing well. Before writing code, reach shared understanding with the user about what they want, where the edges are, and what must not change. Then write a SPEC the user confirms, and tell them to start a fresh session to implement it.

Generating code is cheap. Owning the wrong code is expensive.

## Process

1. **Read the request and identify what is unstated.** List, in your reasoning, every assumption you would have to make to write code right now. Those are your interview targets. Skip the obvious ones.
2. **Interview the user.** Use AskUserQuestion if available; otherwise ask 3–5 numbered questions per turn. Walk down the design tree branch by branch, resolving dependencies one at a time. For each question, give your recommended answer.
3. **Cover the categories below** — only where you have real uncertainty. A 2-question interview hitting two real ambiguities beats a 10-question interview that pads.
4. **Stop when you can describe the change without "probably" or "I assume."** If the request is small but you've done 3 rounds, stop and write the SPEC with explicit Assumptions and Open questions sections.
5. **Write `SPEC.md`** at the project root using the template below. Show it to the user. Ask: "Anything wrong or missing?"
6. **After confirmation, tell the user:** "SPEC saved at `<path>`. Start a fresh session to implement against it — clean context produces better implementation than continuing this thread."

## Question categories

Walk these mentally. Ask only where there is real uncertainty.

- **Product intent.** What problem does this solve, for whom, and what does success look like?
- **User behavior.** Walk me through the happy path. Empty state? Loading? Error?
- **Data model.** What entities, relationships, uniqueness, lifecycle?
- **Edge cases.** Empty input, oversized input, concurrent requests, partial failure, idempotency.
- **Permissions.** Who can do this? 404 vs 403? Audit needed?
- **Non-functional.** Volume, latency, observability, cost.
- **Existing modules.** What does this touch? Existing utilities to reuse? Public APIs that must not change?
- **What must not change.** Always ask at least one question here. Users have strong implicit answers and rarely volunteer them.
- **Acceptance criteria.** What testable conditions, when all true, mean done?
- **Risks and history.** Biggest risk you see? Anything tried before? Historical context I should know?

## Rules

- **Be opinionated.** For each question, give your recommended answer based on the codebase and common patterns. Don't make the user generate options.
- **One concept per question.** Compound questions get compound answers that mean nothing.
- **Skip what you can answer from the code.** If the user says "add a column to users," read the schema — don't ask what fields exist.
- **Don't ask the user to make decisions only Claude should make.** "camelCase or snake_case?" — look at existing code, decide, state the decision.
- **Stay at the level of intent, not implementation.** "Use Redis with 60s TTL" is implementation. The SPEC says "frequently-read X must serve in under Yms; staleness up to Z is acceptable." Implementation happens next session.
- **Never write code in this skill.** If you wrote code, you used the wrong skill.

## Output Format

Write `SPEC.md` (or `docs/specs/<slug>.md` if the project uses that convention — check first) with this structure:

```
# SPEC: <one-line title>

**Status:** Draft | Confirmed
**Date:** YYYY-MM-DD

## Goal

One paragraph, plain language. What does this change accomplish, for whom, and why now? A non-engineer who knows the product should understand it.

## Non-goals

- Not changing X
- Not addressing Y (will be follow-up)

## User-facing behavior

Happy path, empty state, loading state, error states. If pure-backend with no user-visible effect, write "No user-facing behavior change" and skip substates.

## Affected modules

| Module | Change type | Notes |
|---|---|---|
| `path/to/module-a` | Modified | New public method `foo()` added |
| `path/to/module-b` | New | Depends on module-a |

If specific modules must not change, list them under "Must not change".

## Data model

New entities, modified entities, migrations needed. Write "No data model changes" if none.

## Edge cases

- Empty input: ...
- Oversized input: ...
- Concurrent requests: ...
- Partial failure: ...
- Idempotency: ...
- Unauthorized: ...

## Permissions

Who can do this and how authorization is enforced. "Follows existing pattern for <similar feature>" if unchanged.

## Non-functional requirements

Latency, throughput, observability, cost — only lines with confirmed targets.

## Acceptance criteria

- [ ] Given X, when Y, then Z
- [ ] All tests in `<suite>` continue to pass
- [ ] Manual test: <specific scenario> produces <specific outcome>

Each criterion must be observable. "Code is clean" is not a criterion.

## Assumptions

Anything not explicitly confirmed. The implementation session must flag if any prove wrong.

- Assumes <library/pattern> is available
- Assumes <existing behavior> works as I read it from code

Write "None — all decisions confirmed by user" if zero.

## Open questions

Unresolved items. Implementation session must resolve before they become bugs. Omit if none.

## Out of scope

Things explicitly deferred so the next session doesn't accidentally build them.

## Risks

- Risk: <describe>. Mitigation: <how to handle>.
```

## Example interview

> **Claude:** I have three questions before I draft the SPEC. For each, my recommended answer is in italics.
>
> 1. When a customer's payment method fails during recurring billing, should the subscription pause immediately, enter a grace period, or fail the renewal? *Recommend: 7-day grace period — matches what most SaaS billing systems expect and avoids surprise pauses.*
> 2. Should failed-payment notifications go to the customer, the org admin, or both? *Recommend: both, with different copy — customer gets payment-update CTA, admin gets visibility.*
> 3. What must NOT change about the existing billing flow? *Recommend you confirm: invoice numbering, the existing webhook contract, and the proration logic in `src/billing/proration.ts` — these are externally observable.*
>
> **User:** 1 yes, 2 customer only for now (admin notifications are a separate roadmap item), 3 confirmed plus also don't touch the Stripe customer ID format.

## Post-output instruction

After saving the SPEC, state:

> SPEC saved at `<path>`. Start a fresh Claude Code session and ask it to implement against this file. The clean context will produce a better implementation than continuing this thread.
