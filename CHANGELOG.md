# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
the project follows [Semantic Versioning](https://semver.org/).

## Unreleased

- New rule `byte-offset-as-char-index`: a byte offset from
  `String.length`, `String.index-of` or `String.index-of-string` passed
  where `String.prefix`, `String.suffix` or `String.slice` wants a
  character index — directly, through a `let` binding, or with integer
  arithmetic applied on the way. Mixing the two index spaces returns the
  wrong text as soon as one non-ASCII character precedes the offset, and
  an offset past the character count aborts the process. It is reported
  only; the fix is usually `String.byte-slice`, and sometimes a redesign.
  Only the `String.`-qualified spelling is matched, because `Array` has
  same-arity counterparts for all six names. An offset from
  `Pattern.find` is deliberately not reported, and neither is a count
  that is constant by construction, such as `String.length` of an
  all-ASCII string literal.
- The deliberate-discard marker for `unused-let-binding`,
  `shadowed-let-binding` and `unused-defn-parameter` is a leading `-`
  rather than a leading `_`. A leading `_` is still honoured, so nothing
  written against the old spelling starts reporting.
- New rule `unused-defn-parameter`: a `defn` or `defn-` parameter whose
  name occurs nowhere in the body. It is reported only — deleting a
  parameter would break every call site — and prefixing the name with `-`
  silences it. A stub whose body is exactly one of its own parameters is a
  forward declaration and is left alone; the test is the shape, so a
  function that just returns one of its arguments is exempt too.
- `carp-reader` bumped to 0.4.1: a finding on a form that follows a number
  on the same line is now reported at that form's real column. A numeric
  literal advanced the reader's column by one no matter how many bytes it
  occupied, so every column printed after a number on that line was too
  small, and an editor jumping to a finding landed short of it.

## [0.5.1]

- `carp-reader` bumped to 0.4.0: a malformed `\u`, `\U` or `\x` escape in a
  string literal no longer fails the parse, matching the reference reader,
  so angler no longer refuses a file that `carp` compiles and runs.
- A control byte in a string literal is now escaped in a diagnostic's
  `at:` line instead of reaching the terminal raw: `\a`, `\b`, `\t`,
  `\v`, `\f` and `\r` render as those escapes and anything else renders
  as `\uXXXX`, so linting a file that contains one no longer garbles the
  report. A standalone newline is the one exception — it stays raw, so a
  diagnostic quoting a multi-line string literal is still split across
  lines.
- An escape angler does not recognise is now read the way the reference
  compiler reads it — passed through as written — instead of failing the
  whole file with a parse error. Runs of digits after `\` are read as
  hexadecimal character codes, and character literals accept codepoint
  escapes.

## [0.5.0]

- New rule `leaky-top-level-use`, off by default — run it with
  `--only leaky-top-level-use`. A `use` or `use-all` written at a
  file's top level is reported. Carp scopes a `use` to
  the form it is written in, and a file's top level is not a form, so
  the names it brings in reach every file that `load`s this one and can
  turn a bare name there into an ambiguous-symbol error — in a file the
  library's own author never sees. The same `use` inside the `defmodule`
  that needs it is fine. Files nobody can load are exempt: a top-level
  `defn main`, or a `deftest`, a `with-test` or any `assert-…` call
  anywhere in the file, marks a program or a test, where `(use Test)` is
  idiomatic. That test is a heuristic — an example or a sketch driven by
  a framework's own entry point has neither marker and is not a library
  either — which is why the rule is opt-in rather than on by default.
  Reported only, never fixed.
- Rules can be marked off by default with `Lint.mark-opt-in!`, for a
  finding that is a judgement call rather than a defect. An opt-in rule
  is skipped by `Lint.lint` and `Lint.lint-source`, runs under
  `Lint.lint-with`/`Lint.lint-source-with` if the caller's `keep?`
  admits it, and runs from the CLI only when `--only` names it.
  `--list-rules` marks it `[opt-in]`.
- Rules can now be written over a whole file instead of a single node.
  `Lint.register-file-rule!` takes a function from a file's top-level
  forms to its findings, which is what it takes to know whether a form
  sits at top level or to let one part of a file exempt another. Such a
  rule lists, filters and suppresses like any other.
  `Lint.register-rule!` is unchanged.
- Linting a file whose comments hold enough non-ASCII text no longer
  aborts the whole run. A comment such as `; Implementors’ note: …`
  was enough to kill angler before it reported anything, which is why
  it could not lint the Carp compiler's own `core/Derive.carp`.
- Findings can be suppressed in place, from a comment in the linted
  file, instead of only by turning a rule off for the whole run.
  `; angler-disable-next-line` covers the line below the comment,
  `; angler-disable-line` covers the comment's own line and is written
  as a trailing comment, and `; angler-disable-file` covers the whole
  file. Each takes a space- or comma-separated list of rule names, and
  with no names covers every rule. Directives written inside a form
  work, and a directive is matched to a finding by the line the
  offending form starts on. `--fix` obeys them too: a suppressed
  finding is not rewritten, and neither is a form whose rewrite would
  delete a directive along with it. Three new rules watch the
  directives themselves — `unknown-suppression-rule` for one that names
  a rule that does not exist, `unknown-suppression-directive` for a
  comment whose first word is `angler-disable` or opens with
  `angler-disable-` without being one of them, and
  `unused-suppression` for one that suppressed nothing. All three
  answer to `--only` and `--disable` like any other rule, and a
  directive whose rules are already off for the run is never reported
  unused.
- The linter now knows what quoting means. A form under a `quote` is
  data, so no rule reports it and `--fix` never rewrites it: `'(if c
  true false)` used to be rewritten to `'c`, `'(= x true)` to `'x` and
  `'(when (not c) (do a b))` to `'(unless-do c a b)`, each of which
  changes what the surrounding program means. Reports on quoted data
  are gone too, so `'(do x)` no longer reports `lonely-do` and `'(defn
  FooBar [] 1)` no longer reports `non-kebab-case-defn`. A form under a
  `quasiquote` is a template for code rather than data, so it is still
  reported, but it is no longer rewritten either: a spliced body makes
  the arity of the form it lands in unknowable until expansion, and
  `` `(do %@forms) `` is not `` `%@forms ``. The argument of an
  `unquote` or `unquote-splicing` inside a quasiquote is ordinary code,
  and is reported and fixed as such. A hand-written `(quote x)` behaves
  exactly like `'x`, as it does for the compiler.
- Four more boolean simplifications are reported, all on by default.
  `if-bool-branches` rewrites `(if c true false)` to `c` and
  `(if c false true)` to `(not c)`; `not-equal` now covers the other
  direction as well, rewriting `(not (/= a b))` to `(= a b)`. All three
  are fixed by `--fix`: the condition or the operands are evaluated
  exactly once before and after. `eq-self` reports `(= x x)` when both
  operands are the same bare symbol; a call, literal, ref or any other
  compound form on either side is left alone. It is only ever reported,
  never fixed — `(= x x)` is `false` for a NaN float, so folding it to
  `true` would break the idiomatic NaN test.
- New rule `discarded-let-body`, on by default: a plain `let` whose
  body has more than one form is now reported. Carp evaluates the
  first one and drops the rest without compiling, type-checking or
  running them, and says nothing about it — so
  `(let [x (f)] (g x) (h x))` never calls `h`. `--fix` rewrites the
  head to `let-do`, which runs every body form and returns the last.
  That rewrite changes what the code does on purpose; it can also stop
  a file compiling, since `let-do` type-checks the forms `let` never
  looked at. A comment is not a body form, and `let-do` is never
  reported.
- New rule `shadowed-let-binding`, on by default: a `let` or `let-do`
  binding whose name is bound again later in the same vector, with no
  use of it in between, is now reported. `(let [x 1 x 2] x)` computes
  the `1` and throws it away; `(let [x 1 x (+ x 1)] x)` is a deliberate
  pipeline and stays quiet, as does any rebinding whose intervening
  initialisers mention the name. It follows the conventions of
  `unused-let-binding`: names starting with `_` are exempt, a dotted
  symbol segment counts as a use, an odd binding vector is left alone,
  and an `unquote` or `unquote-splicing` anywhere in the form silences
  it. Three or more bindings of one name are one finding. Reported
  only, never fixed — the dead initialiser may be there for its side
  effect.
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
