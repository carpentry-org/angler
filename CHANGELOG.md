# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
the project follows [Semantic Versioning](https://semver.org/).

## Unreleased

- Added `Lint.register-pattern-rule!`: define lint rules as structural
  patterns with `?`-metavariables instead of hand-written match
  functions. Repeated metavariables enforce structural equality.
- Converted built-in rules `lonely-do`, `empty-do`, `double-not`, and
  `set-self` to pattern-based definitions.

## [0.1.1]

- Multi-file CLI invocation (`angler a.carp b.carp …`) no longer
  crashes after the first file. The fix hoists the rule-filter
  closure out of the per-file loop.
- Bumped `carp-reader` dependency to `0.3.0`.

## [0.1.0]

- Initial release.
