<!-- shared:start: edit in christfellowshipchurch/.github, not here -->

## How to work here

- State your assumptions. If a request can be read more than one way, ask before writing code.
- Write the least code that solves the problem: no speculative features, no abstractions used once.
- Change only what the task needs. Don't reformat, rename or refactor neighbouring code.
- Before adding code, read its callers, exports and the shared utilities it should use.
- Match the codebase's conventions. If one looks harmful, say so instead of working around it.
- Tests should fail when the business rule they protect changes.
- Report honestly: never say "done" or "tests pass" if anything was skipped or not run.

## Rules for every app that talks to Rock

- Rock calls, the Rock token and other secrets stay server-side (loaders, actions, `.server.ts`). Never return a secret from a loader or action.
- Never log names, emails, phone numbers, passwords, cookies or OData `$filter` strings. Log ids and timings.
- Validate user input before it goes into a `$filter` string.
- Rock writes and workflow launches are high risk: a redeploy doesn't undo them.

## Risk tiers

- Low: copy, styles, docs, component-only changes.
- Medium: most loader, action and UI feature work.
- High: Rock writes and workflow launches, auth and sessions, cache invalidation, search index config, env files and task definitions, CI workflows.

## Before opening a PR

1. Run this repo's checks (see Commands) and fix what fails.
2. Run /code-review. Fix each finding, or say in the PR why not.
3. Write the PR from the diff against main, using the org PR template (`PULL_REQUEST_TEMPLATE.md` in christfellowshipchurch/.github; `gh api repos/christfellowshipchurch/.github/contents/PULL_REQUEST_TEMPLATE.md -H "Accept: application/vnd.github.raw"`):
   - Risk tier, and which files put it there.
   - Testing: numbered steps for a reviewer who hasn't seen the change, each with its expected result. Say what wasn't tested and why.
   - How to revert.
   - Ticket: search Jira with `assignee = currentUser() AND statusCategory != Done`. If several match or the match is unsure, ask. If none match, write N/A. Never invent a key.
   - A short explainer of the change for the reviewer.
   - The check output.

<!-- shared:end -->
