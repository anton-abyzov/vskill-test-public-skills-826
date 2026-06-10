---
name: git-bisect-detective
version: 1.3.0
description: Drive git bisect to find the exact commit that introduced a regression, including automated test scripts and merge-commit handling.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - git
  - debugging
  - bisect
  - regression
  - ci
---
# git-bisect-detective

Find the precise commit that introduced a bug using `git bisect`. This skill
turns a vague "it worked last week" report into a single offending SHA, then
explains the change.

## When to use

- A test that used to pass now fails and nobody knows which commit broke it.
- A performance regression appeared somewhere in a range of commits.
- You can describe "good" and "bad" states with a repeatable check.

## Core workflow

1. Confirm a reliable reproduction. You need a command that exits `0` when the
   code is good and non-zero when it is bad.
2. Identify a known-good commit (often a release tag) and a known-bad commit
   (usually `HEAD`).
3. Start the bisect session, then let git binary-search the range.

```bash
git bisect start
git bisect bad HEAD
git bisect good v2.4.0        # last release known to work
```

git checks out the midpoint. Test it, then mark the result:

```bash
git bisect good   # the checked-out commit is fine
# or
git bisect bad    # the checked-out commit shows the bug
```

Repeat until git prints `<sha> is the first bad commit`. Always finish with:

```bash
git bisect reset
```

## Automate with a test script

Manual marking is slow and error-prone. Provide a script and let git run it:

```bash
git bisect start HEAD v2.4.0
git bisect run ./scripts/repro.sh
```

`repro.sh` must follow the bisect exit-code contract:

| Exit code | Meaning            | git interprets as |
|-----------|--------------------|-------------------|
| `0`       | commit is good     | `git bisect good` |
| `1`–`124` | commit is bad      | `git bisect bad`  |
| `125`     | cannot test        | `git bisect skip` |
| `>=128`   | abort the bisect   | stop immediately  |

Example script:

```bash
#!/usr/bin/env bash
set -euo pipefail
make build || exit 125          # skip commits that don't compile
npm test -- --run regression.spec.ts
```

## Handling tricky ranges

- **Untestable commits** (broken build, missing deps): return `125` so git runs
  `git bisect skip` instead of mislabeling them.
- **Merge commits**: bisect already understands the DAG; you do not linearize
  history. If a first-bad-commit is a merge, inspect both parents.
- **Flaky tests**: stabilize the repro first. A flaky check makes bisect chase
  noise. Run the test several times locally before trusting a single result.
- **Large ranges**: bisect is `O(log n)`. Even 4,096 commits resolve in ~12
  steps, so do not hand-pick midpoints.

## After you find it

1. `git show <sha>` to read the diff.
2. Summarize *why* the change caused the regression, not just *what* changed.
3. Propose the minimal fix or a revert, and add a regression test so the same
   commit class cannot slip through again.

## Agent guidance

- Never guess the bad commit by reading diffs first — let bisect prove it.
- Always pair a found commit with a new automated test.
- Clean up with `git bisect reset` even when the run fails.
