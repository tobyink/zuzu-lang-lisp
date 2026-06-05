# lang/lisp

Parse and evaluate Lisp/Scheme-style S-expressions from ZuzuScript.

This trial distribution provides a small Scheme-like interpreter and a
general-purpose S-expression parser. Import `lang/lisp` for the facade, or
use `lang/lisp/parser` and `lang/lisp/eval` directly when you only need
one side of the API.

The distribution contains:

- `modules/lang/lisp.zzm`: facade module.
- `modules/lang/lisp/parser.zzm`: S-expression reader and parsed value
  classes.
- `modules/lang/lisp/eval.zzm`: evaluator, environments, callbacks, and
  standard environment.
- `modules/lang/lisp/module`: installable `.zzm` wrappers for packaged
  Lisp source.
- `scripts/lizp`: command-line Lisp runner.
- `tests/lang/lisp`: parser, evaluator, and runner tests.

The distribution is licensed under the Artistic License 1.0 or the GNU
General Public License version 2. See `LICENCE`.

## Parser Values

The parser maps Lisp values onto ordinary Zuzu values where possible:

- proper lists become `Array` values;
- strings become `String` values;
- numbers become `Number` values;
- `#t` and `#f` become booleans;
- symbols become `LispSymbol` objects;
- dotted pairs become `LispPair` chains;
- vector literals become `LispVector` objects;
- character literals become `LispChar` objects;
- bytevector literals become `LispByteVector` objects.

Symbols are objects so a direct Zuzu array can distinguish a Lisp symbol
from a Lisp string literal:

```zzs
from lang/lisp import lisp_eval, standard_env, sym;

say( lisp_eval( [ sym("+"), 1, 2, 3 ], standard_env() ) );  // 6
```

`sym(name)` returns an interned symbol. Use it when constructing
S-expressions directly in ZuzuScript.

## Parsing Source

```zzs
from lang/lisp import parse_program, parse_sexpr, parse_sexprs;

let one := parse_sexpr("(+ 1 2)");
let many := parse_sexprs("(define x 1) (+ x 2)");

let program := parse_program("(define x 1)\n(+ x 2)", "example.lizp");
say( program.location_for( program.get_exprs()[1] ) );  // example.lizp:2:1
```

The reader supports proper lists, dotted pairs, vector literals, bytevector
literals, character literals, numbers, strings, `#t`, `#f`, symbols, `;`
line comments, quote, quasiquote, unquote, and unquote-splicing.

Use `load_sexprs(path)` or `load_program(path)` to read UTF-8 source from
a `std/io` `Path`.

## Evaluation

```zzs
from lang/lisp import lisp_eval_all, parse_sexprs, standard_env;

let env := standard_env();
let result := lisp_eval_all(
    parse_sexprs(
        "(define square (lambda (x) (* x x)))\n" _
        "(square 9)"
    ),
    env,
);
```

`lisp_eval(expr, env?)` evaluates one expression. `expr` may come from the
parser or from a nested Zuzu array using `sym(name)` for symbols.

`lisp_eval_all(exprs, env?)` evaluates expressions from left to right and
returns the final value.

`lisp_repr(value)` formats Lisp values for display.

`core_env()` returns a native core environment. `standard_env()` returns
the same core environment with the Lisp-source prelude loaded. Use
`standard_env({ prelude: false })` when you want core primitives without
prelude definitions.

## Language Support

The evaluator supports:

- lexical environments and closures;
- `quote`, `quasiquote`, `unquote`, and `unquote-splicing`;
- `if`, `begin`, `cond`, `and`, and `or`;
- `define`, function-style `define`, and `set!`;
- fixed, rest-only, and dotted-rest `lambda` parameters;
- `let`, named `let`, `let*`, `letrec`, and `do`;
- `case`;
- `delay` and `force`;
- `guard` and `error`;
- tail-position evaluation through a trampoline;
- `define-library`, `import`, and `include`;
- a practical `syntax-rules` subset with literals, `_`, and `...`.

The standard environment includes arithmetic, numeric comparison,
predicates, equality, list operations, higher-order helpers, vectors,
bytevectors, characters, hash tables, paths, regexps, symbol/string
conversion, string helpers modelled on `std/string`, math helpers modelled
on `std/math`, Scheme-style ports and file I/O, association lookup,
membership lookup, `load`, and `error`.

## Libraries

Libraries use a small R7RS-shaped form:

```scheme
(define-library (demo math)
  (export ten add-ten)
  (import (scheme base))
  (begin
    (define ten 10)
    (define (add-ten x) (+ x ten))))
```

Use `(import (demo math))` from Lisp or `import_library(...)` from
ZuzuScript. Library names such as `(demo math)` resolve to `demo/math.lizp`
under the environment's load paths. If no filesystem source is found, a
relative `.lizp` target also resolves to an installable wrapper module
under `lang/lisp/module`; for example, `scheme/base.lizp` resolves to
`lang/lisp/module/scheme/base`. Import modifiers `only`, `except`,
`prefix`, and `rename` are supported.

The built-in `(scheme base)` library exports the prelude helpers.

## I/O

Lisp-facing I/O uses Scheme-style ports:

```scheme
(call-with-output-file "hello.txt"
  (lambda (p) (display "hello" p)))

(call-with-input-file "hello.txt"
  (lambda (p) (read-line p)))
```

The core I/O surface includes `open-input-file`, `open-output-file`,
`open-append-file`, `read`, `read-line`, `read-char`, `peek-char`,
`display`, `write`, `newline`, close operations, `file-exists?`, and
`delete-file`.

## Diagnostics

Parser errors are `LispSyntaxError` objects with source locations.
Runtime failures are reported as `LispRuntimeError` objects with source
locations and Lisp stack frames where available. `lisp_error_report(error)`
formats a runner-friendly diagnostic report.

## Callbacks

Host code can expose Zuzu callbacks to Lisp code.

Raw callbacks receive the raw S-expression call list, including the
operator symbol. Arguments are not evaluated before dispatch:

```zzs
from lang/lisp import lisp_eval, parse_sexpr, standard_env;

let env := standard_env();
env.define_callback( "foo", function ( call ) {
    return call.length();
} );

say( lisp_eval( parse_sexpr("(foo 1 2 3)"), env ) );  // 4
```

The callback receives a list like:

```zzs
[ sym("foo"), 1, 2, 3 ]
```

Procedure callbacks receive evaluated argument values:

```zzs
env.define_proc_callback( "sum-host", function ( args ) {
    return args[0] + args[1];
} );
```

Environment callbacks receive the raw call list and the current
environment:

```zzs
env.define_env_callback( "lookup-host", function ( call, cb_env ) {
    return cb_env.get(call[1]);
} );
```

## Loading Lisp Files

Use `load_lisp(target, env?)` from ZuzuScript, or `(load "file.lizp")`
from Lisp.

```zzs
from lang/lisp import load_lisp, standard_env;
from std/io import Path;

let env := standard_env();
env.add_load_path("lib");
say( load_lisp( new Path("main.lizp"), env ) );
```

Relative nested loads are resolved from the currently loading file.
Additional load paths are searched after the current base directory. A
recursive load throws an exception.

Packaged Lisp source can be wrapped in a Zuzu module so `zuzuzoo` installs
it along with the rest of the distribution. When `(load "helper.lizp")`
does not find a filesystem file, the loader tries to import
`LISP_SOURCE` from `lang/lisp/module/helper` and evaluates that string as
Lisp source. Nested wrapper loads resolve relative to the wrapper module
path. Filesystem loads take precedence, and wrapper fallback is limited to
safe relative `.lizp` string targets.

## Runner

`scripts/lizp` is executable:

```bash
lizp -e '(+ 1 2 3)'
lizp program.lizp
lizp -I lib program.lizp
```

Options:

- `-e`, `--eval EXPR`: evaluate source passed on the command line;
- `-I`, `--include DIR`: add a Lisp load path;
- `--no-stdlib`: start with an empty environment;
- `--core-only`: start with native core primitives but no prelude;
- `-h`, `--help`: show usage.

For example:

```bash
ZUZULIB=stdlib/modules:tobyink-dists/lang-lisp/modules \
  zuzu tobyink-dists/lang-lisp/scripts/lizp -e '(+ 1 2)'
```
