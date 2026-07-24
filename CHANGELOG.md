# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
the project follows [Semantic Versioning](https://semver.org/).

## Unreleased

- Added `--fix`: rewrite findings in place instead of only reporting
  them. Eight rewrites are available — `lonely-do`, `double-not`,
  `not-equal`, `when-with-not` (both directions), `empty-let-bindings`,
  `eq-true` and `eq-false`. Only the bytes of the fixed forms change,
  so comments and formatting elsewhere in the file are untouched, a
  fix that would lift an `unquote-splicing` form (`%@…`) out of its
  enclosing list is left alone, and a file with nothing to fix is not
  written at all. Rules with no rewrite are still reported, and `--fix`
  exits non-zero if any finding is left over. `--fix --dry-run` prints
  the result instead of writing it.
- `--list-rules` now marks which rules `--fix` can rewrite.
- Added `Lint.fix-source` and `Lint.fix-source-with` for fixing from
  Carp code, plus `Lint.register-fix!`,
  `Lint.register-pattern-fix-rule!` and `Lint.fixable?` for teaching
  new rules how to rewrite.
- Pattern rules loaded from a file can now carry a rewrite template as
  a fourth element: `[<pattern> "name" "message" <template>]`.
- Report a parse error for source containing invalid string escape
  sequences, which were previously accepted silently.
- Fixed `when-with-not` to handle `when-do`/`unless-do` variants (now
  correctly suggests `unless-do`/`when-do` instead of the non-`do`
  counterparts).
- Expanded `non-kebab-case-defn` to also lint `defn-` and `defndynamic`
  names.
- Added six new lint rules: `unsafe-result-unwrap` and `unsafe-maybe-unwrap`
  (pattern-based, flag crash-prone unwrap calls), `eq-true` and `eq-false`
  (redundant boolean comparisons), `cond-single-branch` (single-branch cond
  that should be `if`), and `single-use-let` (`let [x expr] x` is just
  `expr`).
- Added four new built-in lint rules: `identical-if-branches` (`if cond x x`),
  `empty-let-bindings` (`let [] body`), `nested-if-chain` (deeply nested
  if-else suggesting `cond`), and `redundant-do-in-do-variant` (`while-do cond
  (do ...)`).
- Added `Lint.load-rules-from-source!`: load pattern rules from a
  string at runtime. Each rule is an array literal
  `[<pattern> "name" "message"]` — no recompilation needed.
- Added `--rules FILE` CLI flag: load external pattern rules before
  linting. Loaded rules participate in `--only`, `--disable`, and
  `--list-rules` like built-ins.
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
