---
description: >-
  What changed in each released version of the Molecule APIs, newest first, with
  migration notes for anything that breaks an existing integration.
icon: rectangle-history
---

# Release Notes

Each page here tracks one API area. Entries are per released version, newest first, and cover only
what a consumer can observe: contract changes, new and removed operations, and migrations.

| Area | Page |
| -- | -- |
| Labs API | [Labs API](labs-api.md) |
| Tokenization API | [Tokenization API](tokenization-api.md) |
| x402 Gateway | [x402 Gateway](x402-gateway.md) |

**Looking for how to migrate off an older shape?** The
[API Changelog & Migration](../api-reference/changelog.md) page organises the same breaking changes
thematically, by what changed rather than by when. Use this section to answer "what shipped in
1.0.14"; use that one to answer "how do I move off `ipnftUid`".

## What appears here

Only consumer-visible change. Internal refactors, test changes, dependency bumps, infrastructure and
CI work are deliberately absent — most releases contain nothing but those, and produce no entry at
all. A version missing from these pages shipped nothing that affects your integration.

## Entry format

```markdown
## 1.2.3

_Released 2026-08-04_

### Breaking changes

#### <short title>

<what changed, in consumer terms>

**Migration:** <what to do, with a before/after example — or "No action required.">

### Added
### Removed
```

Conventions:

- The heading is the bare version, matching the git tag. Molecule release tags carry **no `v`
  prefix** — the tag for 1.0.14 is `1.0.14`.
- Every breaking change carries either a migration note or an explicit "No action required."
- Link out to the reference page rather than restating it.
