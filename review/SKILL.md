---
name: review
description: This skill should be used when the user asks to "review this branch", "review changes", "review the code", "code review", "review the diff", "check for issues", or wants a thorough code review of changes on any branch. Performs a deep, senior-engineer-level review that goes beyond surface-level checks.
allowed-tools: Bash(git *), Bash(gh *), Grep, Glob, Read
---

# Code Review

Perform a thorough, senior-engineer-level review of changes on a branch. Go beyond linting and surface-level checks. Focus on real issues a human reviewer would flag: design mistakes, inconsistencies, missing error handling, naming problems, convention violations, and logic gaps.

## Workflow

### Step 1: Determine base branch and fetch latest

Determine the base branch this branch will merge into using this priority order:

1. **Check for an open PR**: Run `gh pr view --json baseRefName -q '.baseRefName'`. If a PR exists, use its base branch.
2. **Fall back to `main`**: If no PR exists (command fails), default to `main`.

Store the result in a variable called `BASE` and use it for all subsequent commands.

**Always fetch the latest base branch from the remote before diffing.** A stale local base branch will produce a diff full of already-merged changes, leading to a wrong review.

```
BASE=$(gh pr view --json baseRefName -q '.baseRefName' 2>/dev/null || echo "main")
git fetch origin "$BASE"
```

Then use `origin/$BASE` (not local `$BASE`) for all diff commands to ensure accuracy.

### Step 2: Gather the diff

Run these commands to understand the full scope of changes:

```
git log origin/$BASE...HEAD --oneline
git diff origin/$BASE...HEAD --stat
git diff origin/$BASE...HEAD -- . ':(exclude)package-lock.json' ':(exclude)*.lock' ':(exclude)*.snap'
```

If reviewing a remote branch or PR, fetch it first:
```
git fetch origin <branch>
git diff origin/$BASE...origin/<branch> -- . ':(exclude)package-lock.json' ':(exclude)*.lock' ':(exclude)*.snap'
```

### Step 3: Understand context

Before reviewing line-by-line, understand the intent:
- Read the commit messages to understand what the changes are trying to accomplish
- Identify which parts of the codebase are touched (API, frontend, migrations, tests, etc.)
- Note the scope: is this a new feature, bug fix, refactor, or migration?

### Step 4: Deep review

Review every changed file. For each file, read the surrounding code (not just the diff) to understand how the changes fit into the existing codebase.

Apply every category from the checklist below. Do not skip categories. Cross-reference between files to find inconsistencies.

### Step 5: Verify every finding before reporting

**Every finding must be verified against the actual code on the branch before you include it in the review.** Do not report issues based solely on reading the diff or making assumptions.

Verification rules:
- **"X is unused/dead code"**: Search the ENTIRE codebase on the review branch, not just the changed files. A function might be imported by files outside the PR diff. Use `git grep 'functionName' <branch>` to search all files, not just the ones in the diff.
- **"X is not handled"**: Read the actual file on the branch to confirm the handling is truly missing, not just moved elsewhere in the codebase
- **"Type X is wrong"**: Read the type definition and all its usages across the codebase on the branch to confirm the mismatch
- **"X can never trigger"**: Trace the data flow through the actual code to confirm the condition is unreachable
- **"X falls through to Y"**: Read the full function to confirm the control flow, including early returns
- **Runtime behavior claims** (e.g., "null defeats the fallback"): Reason about JavaScript runtime behavior, not just TypeScript types. A type assertion (`as`) does not change the runtime value - `null as Foo` is still `null` at runtime.

If you cannot verify a finding, do not include it. A smaller review with verified findings is better than a larger review with unverified claims.

### Step 6: Report findings

Present findings grouped by severity. For each issue:
- State the file path and the specific line(s) or code involved
- Explain what the problem is and why it matters
- Suggest a fix if one is obvious

Use this severity scale:
- **Must fix** -- Bugs, data loss risks, security issues, broken rollbacks
- **Should fix** -- Convention violations, design concerns, missing error handling, inconsistencies
- **Suggestion** -- Improvements that are not blocking but would raise code quality

If no issues are found in a category, omit it. Do not pad the review with praise or filler.

**It is perfectly fine for a PR to have no issues.** Only report findings you are confident are genuine bugs, logic errors, or convention violations introduced by this branch. Do not reach for nitpicks, hypothetical edge cases, or style preferences just to fill out the review. If the diff is clean, say so and move on.

## Review Checklist

### Naming and conventions
- Verify camelCase, PascalCase, snake_case used correctly per language/context
- Flag typos in variable names, function names, file names, and string literals
- Check that parameter names accurately describe what they accept (e.g., a param called `userIds` should not actually accept IDs from a different namespace)
- Check file naming conventions: files inside their own folder should typically be named `index.ts` rather than duplicating the folder name
- Verify new route, schema, and Zod object names follow existing patterns in the codebase

### API and architecture patterns
- Check if new endpoints follow existing API conventions (route structure, HTTP methods, middleware patterns)
- Flag when custom one-off routes are created for something the existing API patterns already handle (e.g., bulk-update routes, generic PATCH with schema validation)
- Verify route guards and permissions match the feature requirements
- Check that new patterns do not diverge from established patterns without good reason

### Error handling
- Verify async operations have proper error handling (.catch, try/catch, error boundaries)
- Check that errors are surfaced to the user appropriately
- Flag fire-and-forget promises with no error handling

### Null, undefined, and type safety
- Flag inconsistent null/undefined checks for the same field across files
- Check for optional fields that could be undefined but are only checked against null (or vice versa)
- Verify that type guards and narrowing are correct
- Flag unnecessary type assertions (`as`, `as unknown as`) -- prefer proper typing

### Migrations and database changes
- Verify the down/rollback migration fully reverses everything the up migration does
- If the up migration adds multiple fields, the down migration must remove all of them
- Check for hardcoded values (e.g., lists of document types) that may not be complete or consistent across environments
- Verify indexes are added for fields used in queries

### Frontend conventions
- Flag inline styles that should be in SCSS/CSS files
- Verify the correct icon library is used (check what the rest of the project uses)
- Verify the correct notification/toast system is used (check for existing Notify, toast, or alert utilities before accepting a new library)
- Check for missing memoization on expensive computed values or callbacks passed as props
- Flag functions with long positional parameter lists that should accept an object instead
- Verify consistent use of the project's component library rather than raw HTML or third-party alternatives

### Logic and consistency
- Cross-reference related logic across files -- if the same sorting, filtering, or transformation appears in multiple places, verify they behave identically
- Flag duplicated logic that should be extracted into a shared utility
- Check that new code handles the same edge cases (empty arrays, missing data, concurrent updates) as existing similar code

### UX and design
- Flag redundant UI controls (e.g., both "Back" and "Cancel" buttons doing similar things)
- Check that user-facing text is free of typos
- Verify loading and error states are handled

### Tests
- Check that tests actually test the behavior described in test names
- Flag tests that mock the thing being tested rather than its dependencies
- Verify edge cases are covered (empty inputs, null values, permission boundaries)

### Dependencies
- Flag new dependencies added when existing project utilities already serve the same purpose
- Check that new packages are appropriate in scope and maintained

## Output format

```
## Review: <branch-name>

<1-2 sentence summary of what the changes do>

### Must fix

- **`path/to/file.ts`**: <description of the issue and why it must be fixed>

### Should fix

- **`path/to/file.ts`**: <description of the issue>

### Suggestions

- **`path/to/file.ts`**: <suggestion>
```

Do not use emdashes (--) in the output. Use regular dashes (-) or rewrite the sentence.

If the review is clean, say so briefly. Do not fabricate issues to appear thorough.
