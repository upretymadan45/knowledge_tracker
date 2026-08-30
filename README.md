# Knowledge Tracker

A durable engineering knowledge base captured from technical discussions and learning sessions.

## Repository structure

All knowledge notes must live under the `topics/` directory using this structure:

```text
topics/
  <topic>/
    <concept>.md
```

For example:

```text
topics/
  databases/
    schema-modification-locks.md
    concurrency-strategies.md
```

**Do not create topic directories at the repository root.** `topics/` is the canonical root for knowledge notes.

## Capture trigger

When the user says **"capture this"**, treat it as an explicit instruction to turn the current discussion into a knowledge note and store it in this repository.

The assistant should:

1. Identify the core concept being learned.
2. Remove conversational noise and preserve the actual technical insight.
3. Correct inaccurate assumptions rather than documenting them as facts.
4. Use the standard note format below.
5. Choose the appropriate topic directory under `topics/`.
6. Create or update the relevant Markdown note in that topic directory.
7. Keep notes concise, technically accurate, and production-oriented.
8. Never create a new top-level directory for a knowledge topic.

## Standard note format

```markdown
# <Concept>

## Core idea

<What it is and the essential takeaway.>

## Mental model

<The simplest accurate way to reason about it.>

## Why it exists

<The problem this concept solves.>

## Example

<A concrete example or scenario.>

## Production implications

<How this matters in real systems, including trade-offs and failure modes.>

## Common misconceptions

<Important assumptions that are wrong or incomplete.>

## Related concepts

- <Related concept>
- <Related concept>

## Open questions

- <Questions that remain unresolved, if any>
```

## Principles

- Prefer understanding over documentation dumps.
- Capture mental models and causal relationships.
- Record corrections explicitly when the original assumption was wrong.
- Prefer concrete production examples.
- Avoid unnecessary framework-specific detail unless it affects the concept.
- One concept per note where practical.
- Keep the repository hierarchy predictable: `topics/<topic>/<concept>.md`.
