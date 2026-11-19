# christfellowshipchurch/.github

Shared files for the app repos: cf-external-web, my-church-platform, cf-rev and cf-ocr.

| File | What it is |
| --- | --- |
| `PULL_REQUEST_TEMPLATE.md` | The PR template for every repo in the org |
| `shared/CLAUDE.shared.md` | The master copy of the block at the top of each repo's `CLAUDE.md` |
| `.github/workflows/verify-pnpm.yml` | Reusable checks for the pnpm apps |
| `.github/workflows/verify-bun.yml` | Reusable checks for the Bun apps |
| `.github/workflows/risk-tier.yml` | Reusable check that enforces approvals by risk tier |

This repo is public. Don't put anything here that isn't fine for anyone to read.

## PR template

GitHub uses `PULL_REQUEST_TEMPLATE.md` from this repo for any org repo that has no template of its
own, so the app repos don't keep a copy. A change here shows up in the next PR opened in every repo.

It names no command. The Testing section says to run "this repo's checks (see CLAUDE.md)": each
repo's `CLAUDE.md` lists them under Commands.

`gh pr create` doesn't fill in the org template. To get the text:

```bash
gh api repos/christfellowshipchurch/.github/contents/PULL_REQUEST_TEMPLATE.md -H "Accept: application/vnd.github.raw"
```

## Shared CLAUDE.md block

Each app repo's `CLAUDE.md` has a block between these two lines, identical in every repo:

```
<!-- shared:start: edit in christfellowshipchurch/.github, not here -->
<!-- shared:end -->
```

`shared/CLAUDE.shared.md` is the master copy, markers included. Nothing syncs it automatically.

To change it:

1. Edit `shared/CLAUDE.shared.md` here and merge that PR.
2. In each app repo, on a new branch, replace the block with the master copy:

   ```bash
   gh api repos/christfellowshipchurch/.github/contents/shared/CLAUDE.shared.md \
     -H "Accept: application/vnd.github.raw" > /tmp/CLAUDE.shared.md
   awk -v f=/tmp/CLAUDE.shared.md '
     /^<!-- shared:start/ { while ((getline l < f) > 0) print l; skip = 1; next }
     /^<!-- shared:end -->/ { skip = 0; next }
     !skip
   ' CLAUDE.md > CLAUDE.md.new && mv CLAUDE.md.new CLAUDE.md
   git diff CLAUDE.md
   ```

3. Check the diff touches only the block, then open a PR in that repo. Do all four, so the copies
   stay identical.

## Reusable verify workflows

The org ruleset "Default branch: approval and verify" requires a check named exactly `verify`.
A job that calls a reusable workflow reports as `<caller job> / <called job>`, so the call can't
be the required check itself. Each repo calls the shared workflow in a job named `checks`, then
runs a small job named `verify` that passes only if `checks` succeeded:

```yaml
jobs:
  checks:
    uses: christfellowshipchurch/.github/.github/workflows/verify-pnpm.yml@<full commit SHA> # main
    with:
      test-command: pnpm test

  # The ruleset's required check. `always()` makes it report even when `checks` fails, is
  # skipped or is cancelled, and it fails unless `checks` succeeded. Don't change it to
  # `!cancelled()`: a skipped required check counts as passing.
  verify:
    name: verify
    needs: [checks]
    if: ${{ always() }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Require the shared checks to pass
        env:
          RESULT: ${{ needs.checks.result }}
        run: test "$RESULT" = success
```

Deploy jobs keep `needs: [verify]`.

### Inputs

`verify-pnpm.yml` (checkout, pnpm from `packageManager`, Node from `.nvmrc`, install with
`--frozen-lockfile`, lint, typecheck, format check, test, build):

| Input | Required | Meaning |
| --- | --- | --- |
| `test-command` | yes | Runs the tests once, e.g. `pnpm test` or `pnpm exec vitest run` |

`verify-bun.yml` (checkout, Bun, install with `--frozen-lockfile`, typecheck, optional lint, test,
build):

| Input | Default | Meaning |
| --- | --- | --- |
| `bun-version` | | Bun version, e.g. `1.4.3` |
| `bun-version-file` | | File with the version, e.g. `.bun-version` |
| `lint` | `false` | Also run `bun run lint` |
| `test-command` | `bun run test` | Runs the tests |

Set exactly one of `bun-version` and `bun-version-file`.

### Changing a workflow

The app repos pin the workflow to a commit SHA, so a change here reaches no repo until that repo
moves its pin:

1. Merge the change here.
2. In each app repo, set the `uses:` line to the new commit SHA (`git rev-parse origin/main` here)
   and open a PR. Its `verify` run tests the new version against that repo.

Pin every action in these workflows to a full commit SHA with a version comment.

## Risk tier check

`risk-tier.yml` reads the box ticked under "## Risk tier" in the PR description and decides what
the PR needs before it can merge:

| Tier | Needs |
| --- | --- |
| Low | No approval. Every changed file (and a renamed file's old path) must match the repo's `low-paths`. |
| Medium | An approval of the latest commit from someone other than the author |
| High | The same, plus a code-owner review, which the org ruleset enforces from each repo's CODEOWNERS |

Exactly one box must be ticked. A bot's PR with no box ticked (Dependabot) counts as Medium. A
change request from anyone blocks every tier until it's approved over or dismissed. Approvals from
`ignored-approvers` (default `christfellowshipdev`) don't count.

While a Medium or High PR waits for an approval, the check asks `reviewers` (default `dzwood
leoalfonso14`) to review, so it shows in their queue. It skips the author, drafts, anyone already
asked and anyone who has already reviewed the latest commit. The check still fails until someone
approves; a failed request (Dependabot's token is read-only) only logs a warning.

The org ruleset requires a check named exactly `risk-tier` and 0 approvals, so this check decides.
Each repo calls the shared workflow the same way as verify, and re-runs it when the description
changes or a review comes in:

```yaml
name: Risk tier

on:
  pull_request:
    types: [opened, edited, synchronize, reopened, ready_for_review]
  pull_request_review:
    types: [submitted, dismissed]

permissions:
  contents: read
  pull-requests: write # to request reviewers

concurrency:
  group: risk-tier-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  tier:
    uses: christfellowshipchurch/.github/.github/workflows/risk-tier.yml@<full commit SHA> # main
    with:
      low-paths: |
        docs/**
        **/*.md
        !AGENTS.md

  # The check the org ruleset requires; same pattern as verify.
  risk-tier:
    name: risk-tier
    needs: [tier]
    if: ${{ always() }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Require the risk tier check to pass
        env:
          RESULT: ${{ needs.tier.result }}
        run: test "$RESULT" = success
```

`low-paths` globs: `*` stays within a folder, `**` crosses folders, a line starting with `!`
excludes, and the last matching line wins. Lines starting with `#` are ignored.
