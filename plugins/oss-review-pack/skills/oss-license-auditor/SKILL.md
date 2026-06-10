---
name: oss-license-auditor
version: 1.0.0
description: Audit a project's dependency licenses for compatibility and obligations — copyleft detection, license allowlists, and SPDX-based reporting.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - license
  - compliance
  - open-source
  - spdx
  - audit
---
# oss-license-auditor

Determine whether the licenses of your dependencies are compatible with how you
ship your software, and surface the obligations each one imposes.

## License families

| Family            | Examples                  | Typical obligation                         |
|-------------------|---------------------------|--------------------------------------------|
| Permissive        | MIT, BSD-2/3, Apache-2.0  | keep notice; Apache adds patent grant      |
| Weak copyleft     | MPL-2.0, LGPL-3.0         | share changes to the licensed files        |
| Strong copyleft   | GPL-2.0, GPL-3.0          | derivative works must be GPL               |
| Network copyleft  | AGPL-3.0                  | network use counts as distribution         |
| Public domain     | CC0, Unlicense            | none                                       |

## Audit procedure

1. **Enumerate** every transitive dependency and its declared SPDX license.
   Use the ecosystem's tool: `license-checker` / `npm ls`, `pip-licenses`,
   `cargo-deny`, `go-licenses`.
2. **Normalize** to SPDX identifiers so comparisons are exact (`Apache 2.0` →
   `Apache-2.0`).
3. **Classify** each into the families above.
4. **Compare against your distribution model**:
   - Shipping a proprietary binary? Strong copyleft (GPL) is usually
     incompatible.
   - Running a hosted SaaS? AGPL triggers source-sharing obligations even
     without distributing a binary.
   - Internal-only tool? Obligations are lighter but still tracked.
5. **Report** the obligations: attribution files, notice text, source offers.

## Allowlist / denylist config

```json
{
  "allow": ["MIT", "BSD-2-Clause", "BSD-3-Clause", "Apache-2.0", "ISC"],
  "warn":  ["MPL-2.0", "LGPL-3.0"],
  "deny":  ["GPL-2.0", "GPL-3.0", "AGPL-3.0"],
  "unknownIsError": true
}
```

Fail CI when a `deny` license or an unknown/missing license appears.

## Common pitfalls

- **Dual licensing**: a package may be `MIT OR GPL-3.0` — you may choose the
  permissive option, so record which you rely on.
- **Missing license field**: treat "no license" as *all rights reserved*, not as
  free to use.
- **Bundled code**: a permissive package can vendor a copyleft dependency.
  Inspect transitive trees, not just direct deps.

## Agent guidance

- Always output the *obligation* (attribution, source offer), not just the
  license name.
- Flag unknown/missing licenses as errors, never silently allow them.
- Recommend the least-disruptive replacement when a `deny` license is found.
