# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
the project follows [Semantic Versioning](https://semver.org/).

## Unreleased

- New rule `unused-let-binding`, on by default: a `let` or `let-do`
  binding whose name occurs in none of the initialisers that follow it
  and nowhere in the body is now reported. `set!` counts as a use, as
  does a name that only appears inside a quasiquote or a macro body,
  and as does one segment of a dotted symbol — `Foo.x` keeps `x`
  alive. Bindings whose name starts with `_` are exempt, shadowing is
  not reported, and a binding vector with an odd number of entries is
  left alone. So is a `let` whose bindings or body contain an `unquote`
  or `unquote-splicing` form: an anaphoric macro's body is spliced in at
  expansion time, so a use of the binding need not be in the source at
  all. The rule is only ever reported, never fixed: the initialiser of a
  dead binding may be there for its side effect, which is exactly the
  shape it tends to find.
- `--fix` now rewrites the `-do` half of `when-with-not`: `(when-do (not c)
  …)` becomes `(unless-do c …)` and `(unless-do (not c) …)` becomes
  `(when-do c …)`. Both were reported and never fixed, so `--fix` could
  reach that shape through another rewrite and then stall there —
  `carp`'s own `core/Array.carp` left a `when-with-not` finding behind
  for exactly that reason. A condition that is an unquote-splicing form,
  or a body carrying a comment among the form's own children, is still
  only reported.
- When two rules want to rewrite the same form, `--fix` now always picks
  the same one: the rule registered first. The winner used to depend on
  what else in the file happened to be fixable, so the same construct
  could come out rewritten two different ways in one run.
  `(if c1 () (if c2 y z))` now consistently becomes
  `(unless c1 (if c2 y z))` rather than sometimes `(cond c1 () c2 y z)`.
- `--fix` now rewrites `nested-if-chain`: `(if c1 x (if c2 y (if c3 z w)))`
  becomes `(cond c1 x c2 y c3 z w)`, however deep the chain runs. Every
  branch is moved as the bytes it already was, so multi-line branches
  stay multi-line and keep their comments; run `carp-fmt` afterwards to
  reindent. A comment between a link's condition and its else-branch is
  carried into the `cond` unchanged. A chain is still only reported when
  its inner `if` is malformed or holds a comment before its condition,
  and is left alone entirely when a comment trails the else-branch.
- A metavariable is a `?` followed by one or more name characters and
  nothing else, in rule patterns and rewrite templates alike. Templates
  that name a predicate — say `(unless (empty? ?x) ?y)` — used to report
  their finding and then silently leave the source untouched; `--fix`
  now applies them. A pattern token such as `?*` or `?x?` is an ordinary
  symbol and matches literally, rather than binding a name the template
  resolves to a different value. Affects `--rules` files,
  `Lint.register-pattern-rule!` and `Lint.register-pattern-fix-rule!`.

## [0.4.0]

- `--fix` can now rewrite the findings that restructure a form:
  `form-with-do` (`(let … (do …))` becomes `(let-do … …)`, likewise for
  `while`, `when` and `unless`), `redundant-do-in-do-variant`,
  `if-with-do` (`(if c (do …) ())` becomes `(when-do c …)`, and the
  `unless-do` direction), `if-one-branch-empty` (`(if c x ())` becomes
  `(when c x)`, and the `unless` direction), `cond-single-branch` and
  `empty-do`. The body is spliced in as the bytes it already was, so a
  multi-line body stays multi-line and keeps its comments; only the
  indentation of the spliced lines is left for `carp-fmt` to sort out.
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

## [0.3.0]

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

## [0.2.0]

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
