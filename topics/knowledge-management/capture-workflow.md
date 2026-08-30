# Durable Knowledge Capture Workflow

## Core idea

A conversational learning process becomes much more useful when important technical insights are explicitly captured into a version-controlled knowledge base.

The repository is the durable source of truth. The conversational trigger is **"capture this"**.

When the trigger is used, the current discussion should be transformed into a structured Markdown note and committed to the knowledge repository.

## Mental model

Think of the workflow as:

**Learn → discuss → challenge assumptions → extract insight → structure → commit → retrieve later**

The conversation is the working memory. Git is the durable memory.

The trigger should capture the knowledge at the point where the user decides the discussion is worth retaining.

## Why it exists

Technical knowledge fades when it remains only in conversations or scattered personal notes.

A version-controlled knowledge base provides:

- durable storage
- history through Git commits
- searchable Markdown
- consistent structure
- the ability to refine knowledge over time
- independence from a single conversation

## Example

During a discussion about database concurrency, the user reaches a useful mental model about how the database prevents conflicting updates.

The user says:

> Capture this

The assistant should extract the concept, correct any inaccurate assumptions, place it under the appropriate topic, and commit the resulting Markdown note to the repository.

In a future conversation, the repository can be consulted to recover the established knowledge and continue building on it.

## Production implications

The capture operation should be deterministic and consistent:

1. Treat **"capture this"** as an explicit write command.
2. Determine the actual concept being discussed rather than blindly copying the conversation.
3. Correct technically incorrect statements before storing them.
4. Follow the repository's standard note format.
5. Prefer one coherent concept per note.
6. Check whether a related note already exists before creating duplicates.
7. Update an existing note when the new discussion materially extends or corrects it.
8. Commit the change with a meaningful commit message.
9. Keep the repository readable enough that it remains useful years later.

The workflow should not automatically capture every conversation. Explicit capture prevents the knowledge base from becoming a transcript dump.

## Common misconceptions

- **The conversation itself is durable storage.** It is not a reliable knowledge-management system.
- **GitHub access alone creates a persistent trigger.** The repository can define the protocol, but the assistant still needs to read and follow that protocol in a new conversation.
- **Capturing means copying the transcript.** It should instead extract the durable technical insight.
- **Every discussion deserves a new file.** Existing notes should be extended when the concept already exists.
- **The user's original statement should always be preserved as fact.** Incorrect assumptions should be corrected before capture.

## Related concepts

- Knowledge management
- Version-controlled documentation
- Engineering decision records
- Technical mental models
- Spaced repetition
- Git-based documentation

## Open questions

- Define a lightweight index for fast navigation across topics.
- Decide when two discussions should update the same note versus create separate notes.
- Add metadata such as tags, confidence, and last-reviewed date if the repository grows significantly.
