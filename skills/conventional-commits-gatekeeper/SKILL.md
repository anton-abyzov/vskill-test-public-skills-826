---
name: conventional-commits-gatekeeper
version: 2.0.1
description: Enforce and author Conventional Commits messages, validate commit history, and map commit types to semantic-version bumps.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - git
  - commits
  - conventional-commits
  - semver
  - linting
---
# conventional-commits-gatekeeper

Write, validate, and enforce [Conventional Commits](https://www.conventionalcommits.org)
so that history is machine-readable and releases can be automated.

## The format

```
<type>(<optional scope>)<optional !>: <description>

<optional body>

<optional footer(s)>
```

Example:

```
feat(auth): add refresh-token rotation

Rotates the refresh token on every use to limit replay windows.

Closes #482
BREAKING CHANGE: refresh tokens issued before this release are invalidated.
```

## Allowed types

| Type       | Use for                                          | SemVer bump |
|------------|--------------------------------------------------|-------------|
| `feat`     | a new user-facing feature                        | minor       |
| `fix`      | a bug fix                                         | patch       |
| `perf`     | a performance improvement                         | patch       |
| `refactor` | code change that neither fixes a bug nor adds one| none        |
| `docs`     | documentation only                               | none        |
| `test`     | adding or fixing tests                           | none        |
| `build`    | build system or dependencies                     | none        |
| `ci`       | CI configuration                                 | none        |
| `chore`    | housekeeping with no src/test change             | none        |
| `revert`   | reverts a previous commit                        | varies      |

A `!` after the type/scope **or** a `BREAKING CHANGE:` footer forces a **major**
bump.

## Validation rules

When checking a commit message, fail it if any of these are true:

1. The header does not match `^(\w+)(\([\w/-]+\))?(!)?: .+`.
2. The type is not in the allowed list above.
3. The description starts with a capital letter or ends with a period.
4. The description is in past tense (`added`, `fixed`) — prefer imperative
   (`add`, `fix`).
5. The header exceeds 72 characters.
6. A body line exceeds 100 characters (footers exempt).

## Regex you can reuse

```
^(feat|fix|perf|refactor|docs|test|build|ci|chore|revert)(\([a-z0-9/-]+\))?(!)?: [a-z].{1,70}$
```

## commitlint config

Drop this into `commitlint.config.js` to enforce the same rules in CI:

```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'header-max-length': [2, 'always', 72],
    'subject-case': [2, 'always', 'lower-case'],
    'subject-full-stop': [2, 'never', '.'],
    'type-enum': [2, 'always', [
      'feat','fix','perf','refactor','docs',
      'test','build','ci','chore','revert',
    ]],
  },
};
```

Wire it to a `commit-msg` hook (via husky or `core.hooksPath`) so bad messages
never land.

## Agent guidance

- When asked to commit, infer the smallest correct `type` from the diff, not
  from the user's wording.
- One logical change per commit. If a diff mixes a feature and a fix, split it.
- Put issue references in footers (`Closes #123`), never in the header.
- For releases, aggregate commit types to compute the next version: any `feat`
  → minor, only `fix`/`perf` → patch, any breaking marker → major.
