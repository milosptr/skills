---
name: ubiquitous-language
description: Extract a DDD-style domain glossary from the codebase and conversation, flag ambiguities and synonyms, propose canonical terms, save to UBIQUITOUS_LANGUAGE.md. Use when defining domain terms, hardening terminology, building a glossary, before writing or reviewing code that introduces new vocabulary, or when the user mentions "domain model", "DDD", or "glossary".
---

# Ubiquitous Language

Extract and formalize domain terminology into a consistent glossary so the human, Claude, code, tests, and docs all use the same words for the same things. Save to a local file Claude reads on every relevant invocation.

A messy domain language produces messy code. If `Customer`, `Client`, and `Account` all appear for what is actually one concept, every AI suggestion compounds the mess. This skill picks the canonical word and tells Claude to use it.

## Process

1. **Scan the codebase first.** The codebase is the ground truth — use existing identifiers as the starting vocabulary. Look at: domain folders (`src/domain`, `models/`, `entities/`), schema files, type definitions, exported class and function names. Don't list every variable — pull the **nouns and verbs that appear repeatedly** across modules.
2. **Then scan the conversation** for terms the user uses that may differ from the code.
3. **Identify problems**:
   - Same word used for different concepts (ambiguity)
   - Different words used for the same concept (synonyms)
   - Vague or overloaded terms
   - Terms in code the user never uses (and vice versa) — these are alignment gaps
4. **Propose a canonical glossary** with opinionated term choices. Prefer the term already most used in the code unless the user has a strong reason otherwise.
5. **Write `UBIQUITOUS_LANGUAGE.md`** at the project root using the format below.
6. **Output a summary inline** with the changes proposed, especially renames the user must approve before they propagate.

## Rules

- **Be opinionated.** When multiple words exist for the same concept, pick the best one and list the others as aliases to avoid. Don't ask the user to choose between equivalent options — recommend.
- **Prefer code over conversation when they conflict.** Code is the source of truth. If the user says "client" but the code says `Customer`, recommend `Customer` and note the user's preferred informal term as an alias to avoid.
- **Flag conflicts explicitly.** Every ambiguity goes in the "Flagged ambiguities" section with a clear recommendation.
- **Keep definitions tight.** One sentence max. Define what it IS, not what it does.
- **Show relationships.** Use bold term names and express cardinality where obvious.
- **Only include domain terms.** Skip generic programming concepts (array, function, endpoint) unless they have domain-specific meaning.
- **Group terms into multiple tables** when natural clusters emerge (lifecycle, actors, artifacts). Each group gets its own heading and table. If all terms belong to one cohesive domain, one table is fine — don't force groupings.
- **Include the code identifier.** For each term, record the exact symbol used in code so future sessions name things consistently.
- **Write an example dialogue.** A short conversation (3–5 exchanges) between a developer and a domain expert that uses the terms precisely. The dialogue clarifies boundaries between related concepts.

## Output Format

Write `UBIQUITOUS_LANGUAGE.md` with this structure:

```
# Ubiquitous Language

## Order lifecycle

| Term | Code identifier | Definition | Aliases to avoid |
|------|-----------------|------------|------------------|
| **Order** | `Order` | A customer's request to purchase one or more items | Purchase, transaction |
| **Fulfillment** | `Fulfillment` | The process of preparing and dispatching an Order | Processing, handling |
| **Shipment** | `Shipment` | A physical package dispatched as part of a Fulfillment | Delivery, package |
| **Invoice** | `Invoice` | A request for payment sent after a Shipment is confirmed | Bill, payment request |

## People

| Term | Code identifier | Definition | Aliases to avoid |
|------|-----------------|------------|------------------|
| **Customer** | `Customer` | A person or organization that places Orders | Client, buyer, account |
| **User** | `User` | An authentication identity; may or may not represent a Customer | Login, account |

## Relationships

- An **Order** belongs to exactly one **Customer**
- An **Order** produces one or more **Fulfillments**
- A **Fulfillment** produces one or more **Shipments**
- An **Invoice** is generated when a **Shipment** is confirmed
- A **User** authenticates; a **Customer** transacts — they are distinct

## Example dialogue

> **Dev:** When a **Customer** places an **Order**, do we create the **Invoice** immediately?
> **Domain expert:** No — an **Invoice** is generated only after a **Shipment** is confirmed. A single **Order** can produce multiple **Invoices** if items ship in separate **Shipments**.
> **Dev:** So if a **Shipment** is cancelled before dispatch, no **Invoice** exists for it?
> **Domain expert:** Right. The **Invoice** lifecycle is tied to **Fulfillment**, not to the **Order**.
> **Dev:** And if a **User** isn't tied to a **Customer** — say, an admin viewing reports — they can't place an **Order**?
> **Domain expert:** Correct. The **Customer** relation is what authorizes ordering, not the **User** identity.

## Flagged ambiguities

- "account" was used in conversation to mean both **Customer** and **User**. These are distinct: a **Customer** places Orders, a **User** is an authentication identity that may or may not represent a Customer. Never use "account" in code or new docs.
- "order" appears in code as both `Order` (the entity) and `order` (a noun meaning sequence, e.g., `sortOrder`). Use `Order` only for the entity; rename other uses to `sequence` or `position`.

## Code rename recommendations

Changes the user should confirm before propagating:

- `src/billing/Client.ts` → `Customer.ts` (matches canonical term; `Client` was used in 4 of 47 references)
- `getClientById()` → `getCustomerById()`
```

## Multiple bounded contexts

Evans is explicit: a ubiquitous language belongs to a **bounded context**, not to a project. The same word can mean different things in different contexts and that's healthy — `Customer` in billing (an entity with payment methods) is not the same `Customer` as in support (an identity with a ticket history).

For small, single-context apps a project-wide `UBIQUITOUS_LANGUAGE.md` is fine. For multi-context projects (e.g., billing + auth + reporting + analytics), force the split:

- One glossary per context: `docs/contexts/<context>/UBIQUITOUS_LANGUAGE.md`
- A short root `UBIQUITOUS_LANGUAGE.md` listing the contexts and pointing at each glossary
- A "Cross-context translations" section in the root file when the same noun maps to different concepts across contexts (e.g., billing `Customer` ↔ support `Account`)

If you can't tell which context a term belongs to, that's the signal — surface it to the user as a question, don't pick silently.

## Re-running

When invoked again in the same project:

1. Read the existing `UBIQUITOUS_LANGUAGE.md`
2. Re-scan code and conversation for new terms
3. Update definitions if understanding has evolved
4. Mark changed entries with `(updated)` and new entries with `(new)`
5. Re-flag any new ambiguities
6. Rewrite the example dialogue to incorporate new terms

## Post-output instruction

After writing the file, state:

> I've written/updated `UBIQUITOUS_LANGUAGE.md`. From now on I'll use these terms consistently in code, comments, tests, and docs. If I drift, or if a term should be added, tell me and I'll update the glossary.
