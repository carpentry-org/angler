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

`--fix` obeys `--only` and `--disable`, and files with nothing to fix are
not written at all. Which rules are fixable is marked in `--list-rules`.

## Using it as a library

`angler.carp` is also loadable from Carp code if you want to call the linter
programmatically. The two public entry points are `Lint.lint-source`, which parses and lints a `&String`, and `Lint.lint`, which lints a pre-parsed
`&(Array (Box Form))`.

```clojure
(load "git@github.com:carpentry-org/angler@0.3.0")

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

<hr/>

Have fun!
