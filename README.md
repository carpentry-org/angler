# angler

is a pluggable linter for Carp.

## Install

```bash
git clone git@github.com:carpentry-org/angler
cd angler
carp -b main.carp
# to install
sudo install -m 755 out/angler /usr/local/bin/
```

## Usage

```
angler FILE [FILE...]
```

Each finding is printed on its own line, prefixed with the file path:

```bash
$ angler examples/messy.carp
examples/messy.carp:1:1: [non-kebab-case-defn] defn name 'FooBar' has uppercase letters; Carp convention is kebab-case
  at: (defn FooBar [] 1)
examples/messy.carp:2:1: [form-with-do] (let ... (do ...)) can be (let-do ... ...)
  at: (let [a 1] (do a a))
examples/messy.carp:3:1: [lonely-do] (do x) is just x
  at: (do x)
```

Non-zero exit means something went wrong.

### Fixing

`--fix` rewrites the findings angler can rewrite safely, in place:

```bash
$ angler --fix examples/messy.carp
```

Only the bytes of the fixed forms change — everything else in the file,
comments and formatting included, is left exactly as it was. Findings
whose rule has no fix are printed as usual, and the exit code is zero
only if nothing is left over. `--fix --dry-run` prints the fixed source
to stdout instead of touching the file, and reports the leftovers on
stderr.

Rewrites that restructure a form splice the original bytes of the body
back in, so a multi-line body stays multi-line and its comments stay
where they were. `nested-if-chain` does it for a whole chain at once,
however deep it runs. The indentation of the spliced lines is not
adjusted, though — run `carp-fmt` afterwards if you want the result
reindented.

Some findings are only ever reported, because no rewrite for them is
unconditionally semantics-preserving: renames
(`non-kebab-case-defn`, `non-pascal-case-defmodule`) would need every
call site updated, `identical-if-branches` cannot drop a condition that
may have side effects, `eq-self` cannot fold `(= x x)` to `true` — for a
NaN float it is `false`, which is exactly what that idiom tests for —
`unused-let-binding` and `shadowed-let-binding` cannot delete a binding
whose initialiser may have some, and `unsafe-result-unwrap`,
`unsafe-maybe-unwrap`, `single-use-let` and `try-around-atomic` need to
know more about the program than the linter does.

`eq-self` fires only when both operands are the same bare symbol, so
`(= (f) (f))` and `(= &x &x)` stay quiet: a test suite comparing two
equal values it just built is doing so on purpose, and only the variable
form is reliably a mistake.

`unused-let-binding` reports a `let` or `let-do` binding whose name
occurs in none of the initialisers that follow it and nowhere in the
body. The check is purely syntactic, which keeps it right in places a
scope-accurate one would go wrong: `set!` counts as a use, so does a
name that only appears inside a quasiquote or a macro body, and so does
one segment of a dotted symbol — `Foo.x` keeps `x` alive. Bindings
whose name starts with `_` are never reported, and a binding vector
with an odd number of entries is left alone. Neither is shadowing:
in `(let [x 1 x 2] x)` the second `x` is an occurrence of the name, so
the rule stays quiet. A `let` whose binding vector or body contains an
`unquote` or `unquote-splicing` form is left alone as well — an
anaphoric macro splices its caller's body in at expansion time, so a
use of the binding need not appear in the source at all.

`shadowed-let-binding` covers the hole that leaves behind: a name bound
twice in the same vector, where the first binding is dead. It reports
`(let [x 1 x 2] x)`, in which the `1` is computed and thrown away, but
not `(let [x 1 x (+ x 1)] x)` — there the second initialiser consumes
the first binding, which is a deliberate pipeline rather than a
mistake. Any occurrence of the name between the two bindings keeps the
earlier one alive, including one in the shadowing initialiser itself.
The conventions are the ones `unused-let-binding` uses: a name starting
with `_` is exempt, one segment of a dotted symbol counts as a use, an
odd binding vector is left alone, and an `unquote` or
`unquote-splicing` anywhere in the form silences it. Three or more
bindings of one name are one finding.

`discarded-let-body` reports a plain `let` whose body has more than one
form. Carp's `let` evaluates the first one and returns it; every later
form is dropped without being compiled, type-checked or run, and
neither the compiler nor the reader says a word about it. `while` has
the same rule and rejects the extra forms with an error, and `if` and
`the` reject extra arguments too, so `let` is the one place where the
statements you wrote just vanish. `--fix` rewrites the head to
`let-do`, which runs every body form and returns the last.

That rewrite changes what the code does, which is unusual for a fix
here and is the whole point of this one: the dropped forms are dead
either way, and running them is what the code was written to do. It
can also stop the file compiling, because `let-do` type-checks the
forms `let` never looked at and wants the non-final ones to be `()` —
a loud failure in place of a silent one. Comments are counted as
comments, not body forms, so `(let [x 1] (f x) ; note` stays quiet,
and `let-do` is never reported.

`leaky-top-level-use` reports a `use` or `use-all` written at a file's
top level. Carp scopes a `use` to the form it sits in, and a file's top
level is not a form: the names it brings in are visible in every file
that `load`s this one, where a bare name that resolved before can become
ambiguous. That is a compile error in someone else's file, in a place
the library's own author never sees. The same `use` inside the
`defmodule` that needs it has none of that reach.

Files nobody can load are exempt. A top-level `defn main`, or a
`deftest`, a `with-test` or any `assert-…` call anywhere in the file,
marks a program or a test rather than a library — and `(use Test)` at
the top of a test file is idiomatic. The rule is reported only; where
the `use` should go instead depends on which names the file actually
needs.

`--fix` obeys `--only` and `--disable`, and files with nothing to fix are
not written at all. Which rules are fixable is marked in `--list-rules`.

### Suppressing

`--only` and `--disable` are whole-run switches. To silence one finding
in one place, write a comment next to it:

```clojure
; angler-disable-next-line form-with-do
(let [a 1] (do a a))

(do x) ; angler-disable-line lonely-do

; angler-disable-file non-kebab-case-defn
```

`angler-disable-next-line` covers the line below the comment,
`angler-disable-line` covers the comment's own line — so it is written
as a trailing comment — and `angler-disable-file` covers the whole file
from wherever in it the comment sits. Each takes rule names separated
by spaces or commas; with no names it covers every rule.

A directive is matched to a finding by line, and a finding is reported
on the line the offending form *starts*. So a
`; angler-disable-next-line lonely-do` written above a `(defn f [] …)`
covers the `defn` line, not a `(do x)` three lines into its body. Put
the directive directly above the form that is reported, which is
usually inside another form:

```clojure
(defn f []
  ; angler-disable-next-line lonely-do
  (do x))
```

Suppression covers `--fix` too. A suppressed finding is not rewritten,
and neither is a form whose rewrite would delete a directive along with
it — silently rewriting the code you asked angler to leave alone is the
one thing this feature must not do.

Three rules watch the directives themselves.
`unknown-suppression-rule` fires on a directive that names a rule which
does not exist, `unknown-suppression-directive` on a comment whose
first word is `angler-disable` or opens with `angler-disable-` without
being one of the three, and `unused-suppression` on a directive that
suppressed nothing. They are ordinary rules, so `--disable
unused-suppression` turns the last one off for a run and
`; angler-disable-file unused-suppression` turns it off for a file. A
directive whose rules are already off for the run is never reported
unused, so `--only` and `--disable` do not turn every directive in the
tree into noise.

### Quoting

A form under a `quote` is data, not code, so no rule reports it and
`--fix` never touches it. `'(do x)` is a three-element list a program
may go on to inspect, not a `lonely-do`, and rewriting it to `'x` would
change what the program means. The quote is a latch: everything below
it is data however deep it goes, and an `unquote` under one does not
bring code back — `'(do %x)` really is the list `(do (unquote x))`.

A form under a `quasiquote` is a template that becomes code, so it is
still reported — a `(let … (do …))` in a macro body produces the same
awkward code a hand-written one would. It is never rewritten, though.
A rewrite edits source text, and a splice makes the arity of the form
it lands in unknowable until expansion: `` `(do %@forms) `` looks like
a `lonely-do`, but `%@forms` may splice in nothing or ten forms, and
`` `%@forms `` is the same expansion only in the one-form case. So
findings inside a quasiquote are reported and left for you to judge.

The argument of an `unquote` or `unquote-splicing` inside a quasiquote
is ordinary code again, and is reported and fixed as such, one
quasiquote level per unquote. A `quote` inside a quasiquote is not a
latch — the quasiquote substitutes through it — so its contents stay a
template rather than becoming data.

A hand-written `(quote x)` is treated exactly like `'x`, and likewise
for the other three; the reader expands the punctuation into those
lists, and to the compiler they are the same form.

A suppression directive under a `quote` is read like any other, but
nothing under a quote is reported, so it will be reported unused.

## Using it as a library

`angler.carp` is also loadable from Carp code if you want to call the linter
programmatically. The two public entry points are `Lint.lint-source`, which parses and lints a `&String`, and `Lint.lint`, which lints a pre-parsed
`&(Array (Box Form))`.

```clojure
(load "git@github.com:carpentry-org/angler@0.4.0")

(match (Lint.lint-source "(do x)")
  (Result.Success diags)
    (for [i 0 (Array.length &diags)]
      (IO.println &(Diagnostic.str (Array.unsafe-nth &diags i))))
  (Result.Error e)
    (IO.errorln &(Parser.format-error &e)))
```

`Lint.fix-source` is the fixing counterpart: it takes the same `&String`
and returns the rewritten source.

Rules can carry a rewrite of their own. `Lint.register-pattern-fix-rule!`
takes a template in which each `?name` stands for the original source
text the metavariable matched, and pattern rules loaded with `--rules`
get one from a fourth element:

```clojure
[(identity ?x) "identity-wrap" "identity is a no-op" ?x]
```

Only give a rule a template when the rewrite is unconditionally
semantics-preserving; `--fix` applies it without asking.

### Whole-file rules

`Lint.register-rule!` takes a rule over one node, which is what most
rules need. `Lint.register-file-rule!` takes one over a whole file:

```clojure
(Lint.register-file-rule! @"leaky-top-level-use"
                          @"a use/use-all at the top level of a file others load"
                          rule-leaky-top-level-use)
```

Its function receives the file's top-level forms —
`(Fn [&(Array (Box Located))] (Array Diagnostic))` — and returns every
finding it has. That gives it the two things a node rule cannot have:
it knows whether a form sits at top level, and it can let something
written elsewhere in the file decide whether a form is reported at all.
`leaky-top-level-use` needs both.

A file rule shows up in `--list-rules`, answers to `--only` and
`--disable`, and honours inline `angler-disable` directives like any
other rule. It cannot carry a `--fix`.

<hr/>

Have fun!
