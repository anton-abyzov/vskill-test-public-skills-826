---
name: changelog-scribe
version: 1.0.4
description: Generate and maintain a Keep a Changelog style CHANGELOG.md from Conventional Commits, grouped by type with correct semver headings.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - changelog
  - release
  - documentation
  - semver
  - git
---
# changelog-scribe

Produce a human-readable `CHANGELOG.md` that follows the
[Keep a Changelog](https://keepachangelog.com) convention, derived from commit
history and grouped for release notes.

## Structure

Newest release on top. Keep an `## [Unreleased]` section for work not yet
shipped.

```markdown
# Changelog

All notable changes to this project are documented here.
The format is based on Keep a Changelog, and this project adheres to
Semantic Versioning.

## [Unreleased]

## [1.5.0] - 2026-06-10
### Added
- Refresh-token rotation for the auth service (#482).

### Fixed
- Pagination off-by-one on the search endpoint (#491).

### Security
- Reject expired refresh tokens issued before this release.

[Unreleased]: https://example.com/repo/compare/v1.5.0...HEAD
[1.5.0]: https://example.com/repo/compare/v1.4.0...v1.5.0
```

## Section categories

| Heading      | Source commit types                | Notes                       |
|--------------|------------------------------------|-----------------------------|
| `Added`      | `feat`                             | new capabilities            |
| `Changed`    | `refactor`, `perf`, behavior shifts| existing behavior changed   |
| `Deprecated` | commits noting deprecation         | still works, will be removed|
| `Removed`    | removals                            | gone this release           |
| `Fixed`      | `fix`                              | bug fixes                   |
| `Security`   | `fix` with security impact, CVEs   | call these out explicitly   |

`docs`, `test`, `build`, `ci`, and `chore` commits are normally omitted from
user-facing notes unless they change behavior.

## Building an entry from commits

1. Collect commits since the last tag: `git log v1.4.0..HEAD --pretty='%s%n%b'`.
2. Parse each Conventional Commit header into `(type, scope, description)`.
3. Map the type to a section above.
4. Rewrite the description for a reader who did not see the diff: expand jargon,
   keep it one line, link the PR/issue.
5. Surface `BREAKING CHANGE:` footers prominently — a bold note under the
   version heading and a `### Changed` / `### Removed` entry.

## Choosing the next version

- Any breaking change → bump **major**.
- Otherwise any `feat` → bump **minor**.
- Otherwise (only `fix`/`perf`) → bump **patch**.

## Style rules

- Imperative, present tense: "Add", not "Added a thing" in the bullet body.
- One bullet per user-visible change; group trivial commits.
- Always include the date in `YYYY-MM-DD` and the comparison link footer.
- Never invent entries. If a commit is unclear, ask or read the diff — do not
  guess what it shipped.

## Agent guidance

- Move `## [Unreleased]` content into the new version block on release; leave a
  fresh empty `## [Unreleased]` behind.
- Keep the bottom-of-file link references in sync with each new tag.
- If commits are not Conventional, infer category from the diff but flag that the
  history should adopt Conventional Commits going forward.
