# public-fixture-repo

This is a **vskill E2E test fixture repository**. It exists only to exercise the
vskill skill registry: discovery, validation, security scanning, and plugin
installation flows. The skills and plugin here are intentionally crafted for
testing.

- `skills/` — six genuinely useful PUBLIC (MIT) skills, plus one deliberate
  security-scanner tripwire fixture (`scanner-tripwire-fixture`) that should be
  flagged and **must not be installed**.
- `plugins/oss-review-pack/` — a sample MIT plugin bundling two review skills.

Not intended for production use.
