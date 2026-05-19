# Writing rules for angler

Sometimes you need custom lint rules for your project. This document is for
those moments.

A rule in angler is a Carp function that inspects one AST node at a time and
returns either a diagnostic or nothing. For many rules this means writing a
structural match by hand, but for simple shape-checks there is now a
**pattern language** — see [Pattern-based rules](#pattern-based-rules) below.

## The rule signature

```clojure
(Fn [(Ref Located)] (Maybe Diagnostic))
```

`Located` (from
[the ast library](https://github.com/carpentry-org/carp-reader))
pairs a `Form` with parser-captured position info. Your rule receives one
`Located` per node as the walker visits the AST. Return `(Maybe.Just diag)` to
report a finding, `(Maybe.Nothing)` otherwise. The walker handles recursion
into children, so a rule that wants to fire on a particular shape only needs to
match that shape directly.

## A minimal rule

```clojure
(load "git@github.com:carpentry-org/angler@0.1.0")

(defn rule-todo-comment [l]
  (let [f (Located.form l)]
    (match-ref f
      (Form.Cmt s)
        (if (String.contains-string? s "TODO")
          (Lint.dx (Located.info l)
                   "todo-comment"
                   "comment mentions TODO"
                   (Form.str f))
          (Maybe.Nothing))
      _ (Maybe.Nothing))))
```

`Lint.dx info name message context` is a small helper that packs a `Diagnostic`
and wraps it in `Maybe.Just`. You can also construct `Diagnostic` directly if
you want.

## Registering

Call `(Lint.register-rule! name description fn)`. The simplest place is a
top-level `def`:

```clojure
(Lint.register-rule! @"todo-comment"
                     @"flags any comment that mentions TODO"
                     rule-todo-comment)
```

`name` becomes the bracketed tag in diagnostics (`[todo-comment]`) and the
value matched by `--only` and `--disable` on the command line. `description`
shows up under `angler --list-rules`.

### Caveat on load order

Carp initializes top-level `def`s in dependency-graph order, not in textual or
load order. Your rule may end up registered before or after angler's built-ins,
which affects the order of diagnostics emitted on any one form (but not whether
they fire). If you need deterministic ordering, register from `main` instead of
from a top-level `def`.

## Helpers exposed by `Lint`

These are public, intended for rule authors, and used by every
built-in rule:

| Helper | Purpose |
|---|---|
| `(Lint.nth-form items i)` | extracts the `Form` from the i-th `Located` child of a list's items |
| `(Lint.sym? f name)` | true if `f` is a `Form.Sym` whose dotted name equals `name` (handles `Foo.Bar.baz` style) |
| `(Lint.lst-head? f name)` | true if `f` is a list whose first child is a symbol matching `name` |
| `(Lint.dx info rule msg ctx)` | packages a `Diagnostic` from the given `Info` and wraps it in `Maybe.Just` |

## Walking a compound form

For rules that need to inspect more than just one node (for
instance, checking that a `defmodule` body contains some required
declaration), the walker still feeds you one node at a time. You
can either fire on the outer form and walk its children yourself,
or wait for the walker to visit the inner nodes and decide there.
The latter is usually simpler.

## Multiple findings per node

A rule returns `(Maybe Diagnostic)`, so it fires at most once per
node. If you need to emit several findings on the same form, split
into separate rules. A future minor may add a sibling
`register-multi-rule!` accepting `(Fn [&Located] (Array
Diagnostic))`.

## Pattern-based rules

If your rule just checks the structural shape of a form — "is this a
two-element list whose head is `do`?" — you can skip writing a function
entirely and declare the pattern as a string:

```clojure
(Lint.register-pattern-rule! @"double-not"
                             @"(not (not x)) is just x"
                             "(not (not ?x))"
                             @"(not (not x)) is just x")
```

`register-pattern-rule!` takes four arguments: rule name, description
(for `--list-rules`), the pattern string, and the diagnostic message.

### Metavariables

Any symbol in the pattern that starts with `?` is a **metavariable** —
it matches any single form. If the same `?name` appears more than once,
both positions must match structurally equal forms (compared via
`Form.str`).

Examples:

| Pattern | Matches | Does not match |
|---|---|---|
| `(do ?x)` | `(do foo)`, `(do (+ 1 2))` | `(do)`, `(do a b)` |
| `(set! ?x ?x)` | `(set! a a)` | `(set! a b)` |
| `(not (not ?x))` | `(not (not true))` | `(not true)` |

A bare `?` (single character) is **not** a metavariable — it is treated
as the literal symbol `?`.

### When to use patterns vs. functions

Patterns are great for fixed-shape rules. They cannot express:

- Conditional logic (e.g. "fire only if the name is not kebab-case")
- Variable-length matching (e.g. "a list with 2 *or more* children")
- Cross-node analysis (e.g. inspecting the body of a `defmodule`)

For those, write a function and use `register-rule!`.

## Building your own angler

```bash
git clone git@github.com:carpentry-org/angler
cd angler
# add to main.carp, after the `(load "angler.carp")` line:
#   (load "../my-rules/my-rules.carp")
carp -b main.carp
sudo install -m 755 out/angler /usr/local/bin/
```

Your rules become part of the rule set as if they were built-ins,
controllable via `--only`, `--disable`, and visible under
`--list-rules`.
