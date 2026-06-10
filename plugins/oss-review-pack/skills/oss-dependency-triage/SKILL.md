---
name: oss-dependency-triage
version: 1.0.0
description: Triage third-party dependencies before adoption — maintenance health, security advisories, supply-chain risk, and a go/no-go scoring rubric.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - dependencies
  - security
  - supply-chain
  - open-source
  - risk
---
# oss-dependency-triage

Decide whether to adopt, pin, replace, or remove a third-party dependency. This
skill scores a package on health and risk so the decision is evidence-based.

## Signals to gather

### Maintenance health

- **Release cadence**: regular releases vs. years stale.
- **Open issue/PR ratio and response time**: is anyone home?
- **Bus factor**: one maintainer is a risk; a funded org is safer.
- **Breaking-change discipline**: does it follow semver and keep a changelog?

### Security

- Known advisories: `npm audit`, `pip-audit`, `osv-scanner`, GitHub Advisory DB.
- Time-to-patch on past CVEs.
- Whether it ships a `SECURITY.md` / disclosure process.

### Supply-chain risk

- **Install scripts**: postinstall hooks that run arbitrary code are a red flag.
- **Dependency depth**: a tiny utility pulling 200 transitive deps inflates
  attack surface.
- **Maintainer changes**: recent ownership transfer can precede malicious
  releases — pin and review.
- **Typosquatting**: confirm the exact package name and namespace.

## Scoring rubric

Score each axis 0–2, then sum (max 10):

| Axis              | 0 (bad)                 | 1 (ok)              | 2 (good)                  |
|-------------------|-------------------------|---------------------|---------------------------|
| Maintenance       | abandoned               | sporadic            | active, regular releases  |
| Security history  | unpatched CVEs          | slow patches        | fast patches, clean       |
| Supply chain      | install scripts, deep   | moderate            | no scripts, shallow tree  |
| Popularity/trust  | obscure, 1 maintainer   | moderate adoption   | widely used, org-backed   |
| Fit/alternatives  | no alternative, heavy   | acceptable          | right-sized, swappable    |

| Total  | Decision                                            |
|--------|-----------------------------------------------------|
| 8–10   | adopt; pin to a version range                       |
| 5–7    | adopt with caution; pin exact + monitor advisories  |
| 0–4    | replace or vendor a minimal subset                  |

## Hardening once adopted

- **Pin** versions and commit a lockfile.
- **Verify integrity** via lockfile hashes; enable `npm ci`-style installs.
- **Disable install scripts** where the ecosystem allows (`--ignore-scripts`)
  and audit the few that genuinely need them.
- **Automate** advisory alerts (Dependabot / Renovate) so you learn of CVEs
  without manual polling.

## Agent guidance

- Produce the rubric scores with evidence (links, dates), not vibes.
- Treat postinstall scripts and recent maintainer turnover as elevated risk.
- When recommending removal, name a concrete lighter alternative.
