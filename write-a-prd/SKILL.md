---
name: write-a-prd
description: Create a PRD through user interview, codebase exploration, and module design, then save as an Obsidian note. Use when user wants to write a PRD, create a product requirements document, or plan a new feature.
---

This skill will be invoked when the user wants to create a PRD. You may skip steps if you don't consider them necessary.

1. Ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. Explore the repo to verify their assertions and understand the current state of the codebase.

3. Interview the user relentlessly about every aspect of this plan until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one.

4. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

5. Once you have a complete understanding of the problem and solution, use the template below to write the PRD.

## Obsidian vault structure

Read the vault path from the `OBSIDIAN_VAULT` environment variable. If unset, ask the user for a vault path (and point them at the repo README for how to configure it persistently).

- Create a folder `$OBSIDIAN_VAULT/PRDs/<feature-name>/` for each PRD.
- The PRD itself goes in that folder as `PRD - <feature name>.md`.
- Later, when `/prd-to-slices` is used, slices will be created as sibling files in the same folder, and a dependency graph note will be added. This means the PRD folder becomes a self-contained package for the entire feature.

## PRD template

Use the template below. Note the YAML frontmatter and Obsidian features (tags, callouts).

<prd-template>
---
type: prd
status: draft
created: <today's date YYYY-MM-DD>
tags:
  - prd
  - <feature-area tag, e.g. "auth", "pipeline", "ui">
---

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

## Slices

> [!info] This section is populated by `/prd-to-slices`
> Run `/prd-to-slices` to break this PRD into vertical slices. Links to each slice will appear here.

</prd-template>
