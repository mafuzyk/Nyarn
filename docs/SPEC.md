# Nyarn v1 Language Specification

> **Status:** Design approved; reference interpreter not yet implemented.
>
> **Implementation language:** Go
>
> **Execution model:** Tree-walk interpreter
>
> **Source extension:** `.nyarn`
>
> **License:** MIT

Nyarn is a small interpreted programming language with optional type annotations, a deliberately feline vocabulary, and a runtime that tries very hard to be useful before it becomes dramatic.

The language is intentionally small in v1. Nyarn v1 is single-file, has no classes, records, imports, closures, exceptions-as-catchable-values, formatter, or module system.

The design goal is not to be Python with renamed keywords. Nyarn has its own lexer, parser, AST, semantic checker, runtime, call stack, diagnostics, and execution model.

---

## 1. Design principles

Nyarn v1 follows these rules:

1. **Small language, real semantics.** The joke is in the vocabulary, not in undefined behavior.
2. **Useful errors first, cat panic second.** Diagnostics must be actionable before they become unserious.
3. **No implicit magic unless explicitly specified.** Numeric promotion is narrow and predictable; string/number coercion is not automatic.
4. **Immutability is the default.** `mew` creates immutable bindings; `scratch` creates mutable bindings.
5. **Optional typing should help, not punish.** Inference is the default. Explicit annotations create enforceable contracts.
6. **The v1 runtime stays simple.** Tree-walk interpretation, lexical scopes, top-level functions, no closures.
7. **YAGNI.** Features such as modules, records, catchable errors, formatter support, and richer standard library operations are reserved for later versions.

---

## 2. Source files and basic syntax

Nyarn source files use the `.nyarn` extension.

Statements are terminated by a newline. Semicolons are not used.

```nyarn
mew nyame = "Nyarn"
scratch lives: paw = 9
purr "Henlo from {nyame}!"
```

Blank lines are ignored.

Inside grouping constructs such as `(...)` and `[...]`, line breaks may appear freely without terminating the surrounding expression.

```nyarn
mew result = (
    10 +
    20 +
    30
)

mew cats: clowder meow = [
    "Milo",
    "Luna",
    "Dexter"
]
```

### 2.1 Comments

Nyarn v1 has single-line comments only.

```nyarn
# this is a comment
mew nyame = "Mafu"  # inline comment
```

There is no multiline comment syntax in v1.

### 2.2 Blocks

Blocks use braces.

```nyarn
sniff hungry {
    purr "FOOD"
}
```

Every `{ ... }` block creates a lexical child scope.

---

## 3. Identifiers and keywords

Nyarn identifiers are case-sensitive.

```text
nyame
Nyame
```

These are distinct names.

Identifiers may contain Unicode letters, digits after the first character, and `_`. Emoji are not valid identifier characters.

Examples:

```nyarn
mew gatinho = "mrrp"
mew coração = purring
mew cat_2 = "Luna"
```

Keywords cannot be used as identifiers.

`nyame` is **not** a keyword. It is merely Nyarn culture.

### 3.1 v1 keywords

```text
mew
scratch
sniff
peek
nap
hunt
in
leap
prowl
pounce
gift
purr
hiss
listen
andpaw
orpaw
nah
paw
whisker
meow
mood
clowder
maybe
purring
hissing
emptybowl
stash
snatch
count
sniffout
```

### 3.2 Reserved for future versions

The following names are reserved in v1 and cannot be used as identifiers:

```text
pspsps
breed
carefully
hissback
```

If used as syntax, the parser should produce a clear “reserved/not yet supported” diagnostic rather than silently treating the token as an identifier.

Planned meanings:

```text
pspsps     future imports/modules
breed      future record/struct-like declarations
carefully  future error-catching construct
hissback   future error handler
```

---

## 4. Types

Nyarn v1 exposes the following types:

```text
paw        integer
whisker    floating-point number
meow       string
mood       boolean
clowder T  homogeneous list of T
maybe T    optional T; may also contain emptybowl
```

Special values:

```text
purring    true
hissing    false
emptybowl  null / absence
```

### 4.1 Type annotations

Type annotations are optional.

```nyarn
mew age = 17
mew height: whisker = 1.72
mew nyame: meow = "Mafu"
mew sleepy: mood = purring
```

When an annotation is present, it is a contract enforced by the checker.

```nyarn
mew age: paw = "seventeen"
```

This is a type error.

### 4.2 Type inference

Literal inference:

```text
17              -> paw
1.72            -> whisker
"Mafu"          -> meow
purring          -> mood
hissing          -> mood
[1, 2, 3]       -> clowder paw
[1, 2.5]        -> clowder whisker
["a", "b"]     -> clowder meow
```

A heterogeneous list is invalid:

```nyarn
mew chaos = [1, "cat", purring]
```

A bare empty list cannot be inferred:

```nyarn
mew cats = []
```

The programmer must annotate it:

```nyarn
mew cats: clowder meow = []
```

### 4.3 `maybe` and `emptybowl`

`emptybowl` is only assignable to `maybe T`.

Valid:

```nyarn
mew nickname: maybe meow = emptybowl
mew age: maybe paw = 17
```

Invalid:

```nyarn
mew nickname: meow = emptybowl
```

`emptybowl` is not truthy/falsy and does not implicitly convert to another type.

### 4.4 Type narrowing

A comparison against `emptybowl` narrows a `maybe T` within the relevant branch.

```nyarn
mew nickname: maybe meow = emptybowl

sniff nickname != emptybowl {
    # nickname is meow here
    purr "Henlo, {nickname}"
}
```

The inverse is also recognized:

```nyarn
sniff nickname == emptybowl {
    # nickname is known to be emptybowl here
}
nap {
    # nickname is meow here
}
```

The narrowing applies to the lexical branch where the check proves the value state.

---

## 5. Variables and mutability

### 5.1 `mew`

`mew` declares an immutable binding.

```nyarn
mew nyame = "Mafu"
```

Reassignment is forbidden.

```nyarn
mew nyame = "Mafu"
nyame = "Nyarn"  # HISS
```

For v1 composite values, immutability applies to mutation through the binding as well.

```nyarn
mew cats = ["Milo", "Luna"]
cats[0] = "Dexter"  # HISS
```

### 5.2 `scratch`

`scratch` declares a mutable binding.

```nyarn
scratch lives: paw = 9
lives = 8
```

Mutable `clowder` contents may also be changed.

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]
cats[0] = "Dexter"
```

### 5.3 Where mutability lives

Mutability is a property of the binding, not of the runtime value itself.

The runtime therefore checks the root binding before any list mutation in v1.

### 5.4 Shadowing and redeclaration

Shadowing is allowed in child scopes:

```nyarn
mew nyame = "Mafu"

sniff purring {
    mew nyame = "Nyarn"
    purr nyame  # Nyarn
}

purr nyame      # Mafu
```

Redeclaring the same name in the same scope is forbidden.

---

## 6. Boolean logic and conditions

Nyarn has no general truthiness.

Only a `mood` value may be used as a condition.

Valid:

```nyarn
mew hungry: mood = purring

sniff hungry {
    purr "feed me"
}
```

Invalid:

```nyarn
mew lives = 9

sniff lives {
    purr "nope"
}
```

Logical operators:

```text
andpaw   logical AND
orpaw    logical OR
nah      logical NOT
```

Example:

```nyarn
sniff hungry andpaw nah sleepy {
    purr "time to destroy the kitchen"
}
```

Logical operators require `mood` operands and produce `mood`.

---

## 7. Control flow

### 7.1 `sniff`, `peek`, `nap`

Nyarn conditional chains use:

```text
sniff  if
peek   else if
nap    else
```

Example:

```nyarn
sniff hungry {
    purr "FOOD"
}
peek sleepy {
    purr "zzz"
}
nap {
    purr "whatever"
}
```

A chain may contain zero or more `peek` branches and at most one final `nap` branch.

Every condition must evaluate to `mood`.

### 7.2 `hunt`

`hunt` has two forms.

Conditional loop:

```nyarn
hunt lives > 0 {
    lives = lives - 1
}
```

Iteration loop:

```nyarn
hunt cat in cats {
    purr cat
}
```

The iteration variable is immutable, equivalent to a `mew` binding inside the loop body.

`hunt ... in ...` accepts:

- `clowder T`
- `meow`
- range values

String iteration operates on Unicode characters, not raw bytes.

### 7.3 `leap`

`leap` exits the nearest enclosing `hunt`.

```nyarn
hunt cat in cats {
    sniff cat == "dangerous" {
        leap
    }
}
```

Using `leap` outside `hunt` is a checker error.

### 7.4 `prowl`

`prowl` continues with the next iteration of the nearest enclosing `hunt`.

```nyarn
hunt cat in cats {
    sniff cat == "annoying" {
        prowl
    }

    purr cat
}
```

Using `prowl` outside `hunt` is a checker error.

---

## 8. Ranges

Nyarn v1 supports half-open integer ranges.

```nyarn
0..10
```

means `0` through `9`.

Example:

```nyarn
hunt i in 0..3 {
    purr i
}
```

Output:

```text
0
1
2
```

Range bounds are `paw` values in v1.

There is no custom range step syntax in v1.

The runtime should represent a range lazily rather than materializing a `clowder`.

---

## 9. Functions

Functions are declared with `pounce`.

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}
```

### 9.1 Parameters

Parameter annotations are optional.

```nyarn
pounce greet(nyame) {
    purr "Henlo, {nyame}"
}
```

An unannotated parameter is dynamically typed for checker purposes.

The checker may represent this internally with an `any`-like type, but `any` is not a user-visible Nyarn v1 type.

Nyarn requires the exact number of arguments declared by the function.

Default arguments are not supported in v1.

### 9.2 Return values

`gift` returns from a function.

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}
```

`gift` with no expression is allowed and returns `emptybowl`.

```nyarn
pounce stop() {
    gift
}
```

Falling off the end of a function also returns `emptybowl`.

Return annotations are optional.

```nyarn
pounce weird(a, b) {
    gift a + b
}
```

If a return annotation exists, every reachable `gift` must match it.

If a function has a non-`maybe` explicit return type, the checker must reject a body that can reach the end without a compatible `gift`.

Invalid:

```nyarn
pounce get_life(x: paw): paw {
    sniff x > 0 {
        gift x
    }
}
```

Valid:

```nyarn
pounce get_life(x: paw): maybe paw {
    sniff x > 0 {
        gift x
    }
}
```

### 9.3 Top-level only

`pounce` declarations are only valid at top level in v1.

Nested functions are not supported.

### 9.4 Hoisting

Top-level `pounce` declarations are registered before top-level statements execute.

Therefore this is valid:

```nyarn
greet("Mafu")

pounce greet(nyame: meow) {
    purr "Henlo, {nyame}"
}
```

Only functions are hoisted. `mew` and `scratch` bindings do not exist before execution reaches their declaration.

### 9.5 Recursion

Recursion is supported in v1.

```nyarn
pounce factorial(n: paw): paw {
    sniff n <= 1 {
        gift 1
    }

    gift n * factorial(n - 1)
}
```

### 9.6 Globals

Functions may read global bindings.

Functions may modify a global binding only when it was declared with `scratch`.

### 9.7 No closures

Nyarn v1 does not support closures.

Functions do not capture local bindings from enclosing function calls because nested `pounce` declarations do not exist in v1.

---

## 10. Numbers and operators

Arithmetic operators:

```text
+  -  *  /  %  **
```

Comparison operators:

```text
==  !=  >  >=  <  <=
```

Assignment:

```text
=
```

### 10.1 Numeric promotion

Nyarn has exactly one implicit numeric promotion in v1:

```text
paw -> whisker
```

Examples:

```text
paw + paw         -> paw
paw + whisker     -> whisker
whisker + paw     -> whisker
whisker + whisker -> whisker
```

The same promotion rule applies to numeric comparisons.

### 10.2 Division

`/` always returns `whisker`.

```nyarn
purr 5 / 2
# 2.5
```

Division by zero is a runtime error.

Integer division syntax is not included in v1.

### 10.3 Modulo

`%` is supported in v1 for numeric operands.

When both operands are `paw`, the result is `paw`.

If either operand is `whisker`, normal numeric promotion applies and the result is `whisker`.

Modulo by zero is a runtime error.

### 10.4 Exponentiation

`**` is supported in v1 and is right-associative.

```nyarn
2 ** 3 ** 2
```

is parsed as:

```text
2 ** (3 ** 2)
```

If both operands are `paw` and the mathematical result is exactly representable as an integer under the implementation's checked integer rules, the result is `paw`; otherwise the result is `whisker`.

### 10.5 String concatenation

`+` concatenates two `meow` values.

```nyarn
purr "nya" + "rn"
# nyarn
```

Nyarn does not automatically concatenate strings with numbers or other types.

```nyarn
purr "lives: " + 9
```

is a type error.

Use interpolation or explicit conversion instead.

### 10.6 Equality across incompatible types

Comparing incompatible types with `==` or `!=` is a type error, not merely `hissing`.

```nyarn
5 == "5"
```

is invalid.

Numeric equality permits the `paw -> whisker` promotion.

```nyarn
5 == 5.0
# purring
```

### 10.7 Ordering

Numeric ordering is supported with numeric promotion.

`meow` values support lexical ordering:

```nyarn
"Amora" < "Dexter"
```

`mood` supports `==` and `!=` only.

`clowder` supports `==` and `!=` by structural comparison only.

`clowder` does not support ordering in v1.

`emptybowl` may only be compared using `==` or `!=` in a valid `maybe` context.

### 10.8 Chained comparisons

Chained comparison syntax is not supported in v1.

Invalid:

```nyarn
0 < age < 18
```

Use:

```nyarn
age > 0 andpaw age < 18
```

---

## 11. Strings

Nyarn v1 uses double-quoted strings only.

```nyarn
mew nyame = "Mafu"
```

Supported escapes:

```text
\n
\t
\"
\\
```

### 11.1 Interpolation

Strings support expression interpolation with `{ ... }`.

```nyarn
mew nyame = "Mafu"
mew lives = 9

purr "{nyame} has {lives} lives"
purr "next life: {lives - 1}"
```

Interpolation accepts full expressions.

Literal braces use doubled braces:

```nyarn
purr "{{hello}}"
```

which displays:

```text
{hello}
```

Interpolation formats the runtime value for display; it does not mutate the value's type.

---

## 12. Lists: `clowder`

A `clowder T` is a homogeneous list of `T`.

```nyarn
mew cats: clowder meow = ["Milo", "Luna", "Dexter"]
mew numbers: clowder paw = [1, 2, 3]
```

### 12.1 Indexing

Indexing uses `[...]`.

```nyarn
purr cats[0]
```

Negative indexing is supported in Python-like style.

```nyarn
purr cats[-1]
```

returns the last element.

An out-of-range index is a runtime error.

It does not return `emptybowl`.

### 12.2 Mutation

Indexed assignment requires a mutable root binding.

```nyarn
scratch cats = ["Milo", "Luna"]
cats[0] = "Dexter"
```

The assigned element must be compatible with the `clowder` element type.

### 12.3 Structural equality

Two compatible `clowder` values may be compared with `==` or `!=`.

```nyarn
purr [1, 2] == [1, 2]
# purring
```

Comparison is element-by-element and order-sensitive.

### 12.4 Slicing

List slicing is not included in v1.

---

## 13. Core built-ins

Nyarn v1 has a deliberately tiny set of built-ins.

These are language-level core operations, not object methods and not a module system.

### 13.1 `purr`

`purr` is a statement that evaluates exactly one expression, displays it, and appends a newline.

```nyarn
purr "hello"
purr 42
purr purring
```

Display formatting is not the same operation as converting a value to `meow`.

### 13.2 `listen`

`listen` is an expression that returns `meow`.

With prompt:

```nyarn
mew nyame = listen "Nyame: "
```

Without prompt:

```nyarn
mew input = listen
```

If a prompt is supplied, it must be `meow` and is displayed without an automatic newline before reading stdin.

The trailing input newline is removed.

### 13.3 Type conversions

The names of base types may be used as explicit conversion functions.

```nyarn
paw("17")
whisker("1.72")
meow(42)
mood("purring")
```

Required v1 behavior:

- `paw(paw)` returns the same value.
- `paw(whisker)` is allowed only when the value is finite and exactly integral; otherwise HISS.
- `paw(meow)` parses a valid base-10 integer representation; otherwise HISS.
- `whisker(paw)` performs exact numeric promotion where representable.
- `whisker(whisker)` returns the same value.
- `whisker(meow)` parses a valid floating-point representation; otherwise HISS.
- `meow(x)` returns the canonical Nyarn textual representation of a supported runtime value.
- `mood(mood)` returns the same value.
- `mood(meow)` accepts exactly `"purring"` and `"hissing"`; any other string HISSes.
- No numeric-to-`mood` conversion exists in v1.
- `emptybowl` cannot be converted to a non-`maybe` base type.

### 13.4 `stash`

`stash` appends an item to a mutable `clowder`.

```nyarn
scratch cats: clowder meow = ["Milo"]
stash cats, "Amora"
```

The first operand must resolve to a mutable `clowder` binding.

The value must be compatible with the element type.

`stash` is a statement in v1 and does not produce a useful value.

### 13.5 `snatch`

`snatch` removes and returns an element from a mutable `clowder`.

Without index, it removes the final element:

```nyarn
mew last = snatch cats
```

With index:

```nyarn
mew removed = snatch cats, 2
```

Negative indices are supported using the same indexing rules as normal list access.

Calling `snatch` on an empty list or with an invalid index is a runtime error.

The first operand must resolve to a mutable `clowder` binding.

### 13.6 `count`

`count` returns a `paw` length.

```nyarn
purr count cats
purr count "mrrp"
```

For `meow`, length counts Unicode characters, not bytes.

For `clowder`, length counts elements.

### 13.7 `sniffout`

`sniffout` tests membership and returns `mood`.

```nyarn
purr sniffout cats, "Amora"
purr sniffout "Nyarn", "yar"
```

For `clowder`, it performs element membership using Nyarn equality rules.

For `meow`, it checks whether the second `meow` occurs as a substring.

Unsupported operand types are checker errors when known statically and runtime errors when only discoverable dynamically.

---

## 14. `hiss` and runtime failure

`hiss` raises a Nyarn runtime error.

```nyarn
sniff lives < 0 {
    hiss "cats cannot have negative lives"
}
```

In v1, `hiss` cannot be caught. It propagates to top level and terminates file execution with a nonzero exit code.

The runtime architecture should nevertheless represent it as a dedicated control/error signal so future `carefully` / `hissback` support can be added without redefining `hiss` semantics.

---

## 15. Runtime model

The reference implementation uses a tree-walk interpreter in Go.

Pipeline:

```text
source
  -> lexer
  -> tokens
  -> parser
  -> AST
  -> semantic checker
  -> interpreter
  -> runtime values / environments / call stack
```

Normal file execution does not begin until lexing, parsing, and semantic checking succeed.

### 15.1 Runtime values

The implementation should have distinct runtime representations for at least:

```text
PawValue
WhiskerValue
MeowValue
MoodValue
ClowderValue
EmptyBowlValue
FunctionValue
RangeValue
```

`FunctionValue` and `RangeValue` are implementation/runtime concepts, not public v1 type keywords.

### 15.2 Environments

Each lexical block creates a child environment.

Conceptually:

```text
Global
  -> Function call
       -> Sniff/Hunt/other block
```

A binding tracks at minimum:

```text
name
value
declared/inferred type
mutable flag
source information as useful for diagnostics
```

Name lookup walks outward through lexical parents according to Nyarn scope rules.

Because v1 has no closures, a function call environment may access its own locals/parameters and globals, but there is no captured enclosing function-local environment.

### 15.3 Control signals

The interpreter should use dedicated internal control signals for:

```text
ReturnSignal
BreakSignal
ContinueSignal
HissSignal
```

This prevents control flow from being modeled through ad-hoc booleans passed through every evaluator method.

### 15.4 Function calls

A function call conceptually:

1. Resolves the function.
2. Validates arity.
3. Creates a call frame.
4. Creates the function environment.
5. Binds parameters.
6. Executes the body.
7. Captures `gift` as `ReturnSignal`.
8. Returns `emptybowl` if the body ends normally.
9. Removes the frame/environment.

### 15.5 Top-level execution and hoisting

File execution occurs in two logical passes after successful checking:

```text
Pass 1: register all top-level pounce declarations
Pass 2: execute non-declaration top-level statements in source order
```

Top-level variable declarations execute in source order and are not hoisted.

### 15.6 Range execution

Ranges should be lazy runtime values rather than prebuilt lists.

### 15.7 Go panics

Ordinary Nyarn user errors must never leak raw Go panic output.

An unexpected interpreter bug should produce a distinct internal diagnostic such as:

```text
NYARN INTERNAL ERROR

The interpreter tripped over its own yarn.
This is probably a Nyarn bug, not yours.
```

Raw Go stack traces should only be available in an explicit debug mode such as `NYARN_DEBUG=1`.

---

## 16. Diagnostics

Nyarn diagnostics follow one rule:

> **Useful first. Unhinged second.**

A diagnostic should carry enough structured data to support:

```text
kind
file
line
column
source span
primary message
optional hint
optional cat message
optional Nyarn stack trace
```

### 16.1 Diagnostic categories

```text
HISS [syntax]
HISS [type]
HISS [runtime]
HISS [name]
HISS [io]
HISS [internal]
```

### 16.2 Example: type error

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

8 | sniff lives {
          ^^^^^

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

### 16.3 Example: unknown name

```text
HISS [name] at main.nyarn:4:6
unknown identifier `nymae`

4 | purr nymae
         ^^^^^

did you mean `nyame`?

THE CAT FOUND NO SUCH THING UNDER THE COUCH.
```

The checker should offer typo suggestions for nearby identifiers when the similarity is strong enough to be useful.

### 16.4 Example: immutable binding

```text
HISS [type] at main.nyarn:5:1
cannot modify `cats`
`cats` was declared with `mew`

5 | cats[0] = "Dexter"
    ^^^^^^^

THE CAT HAS BEEN TOLD NOT TO SCRATCH THIS.
```

### 16.5 Example: invalid loop control

```text
HISS [syntax] at main.nyarn:2:1
`leap` can only be used inside `hunt`

2 | leap
    ^^^^

WHERE ARE YOU LEAPING TO???
THERE IS NO HUNT.
```

### 16.6 Example: bounds error

```text
HISS [runtime] at main.nyarn:9:13
clowder index 8 is out of bounds
length: 3

9 | purr cats[8]
               ^

THE CAT RAN PAST THE END OF THE YARN.
THERE IS NOTHING THERE.
```

### 16.7 Example: deep recursion

A future implementation-defined maximum call depth may produce:

```text
HISS [runtime]
maximum call depth exceeded

THE YARN IS TOO TANGLED!!!
IS THE PROGRAM RECURSING?!
CAN'T KEEP UP!!!
```

The exact maximum depth is an implementation limit, not a language-level semantic guarantee.

### 16.8 Stack traces

Runtime errors include Nyarn call frames when available.

```text
HISS [runtime] at calc.nyarn:14:9
division by zero

  at divide()       calc.nyarn:14
  at calculate()    calc.nyarn:22
  at main_logic()   main.nyarn:8
  at <top-level>    main.nyarn:13

THE CAT HAS KNOCKED THE CALL STACK OFF THE TABLE.
```

---

## 17. CLI

The reference CLI accepts both direct execution and an explicit `run` subcommand.

```bash
nyarn hello.nyarn
nyarn run hello.nyarn
```

Both execute the same Nyarn source file.

Running `nyarn` with no file starts the REPL.

### 17.1 Exit codes

The reference CLI uses:

```text
0  success
1  runtime failure / explicit hiss
2  syntax/parser failure
3  type/checker failure
4  CLI, file, or input/output failure
```

Internal interpreter failures should also be nonzero and clearly marked as internal errors.

---

## 18. REPL

Running:

```bash
nyarn
```

starts an interactive session.

Example:

```text
Nyarn REPL
> mew x = 9
> purr x
9
> scratch y = 2
> y = 7
> purr y
7
```

The global REPL environment persists across entries.

Top-level functions declared in the REPL also persist.

The REPL accepts multiline input when braces, brackets, or parentheses are incomplete.

```text
> pounce greet(nyame: meow) {
...     purr "Henlo, {nyame}"
... }
>
```

A user error does not terminate the REPL session.

```text
> purr 1 / 0

HISS [runtime]
division by zero

THE CAT HAS ATTEMPTED FORBIDDEN MATHEMATICS.

>
```

Each submitted unit is lexed, parsed, checked, and then interpreted against the persistent REPL environment.

---

## 19. Grammar

The grammar below is normative in intent. Concrete parser implementation details may differ as long as accepted programs and precedence match this specification.

```text
program           -> statement* EOF

statement         -> declaration NEWLINE?
                   | function_decl NEWLINE?
                   | if_statement NEWLINE?
                   | hunt_statement NEWLINE?
                   | stash_statement NEWLINE?
                   | purr_statement NEWLINE?
                   | gift_statement NEWLINE?
                   | hiss_statement NEWLINE?
                   | leap_statement NEWLINE?
                   | prowl_statement NEWLINE?
                   | expression_statement NEWLINE?

declaration       -> ("mew" | "scratch") IDENTIFIER type_annotation? "=" expression

type_annotation   -> ":" type

type              -> "paw"
                   | "whisker"
                   | "meow"
                   | "mood"
                   | "clowder" type
                   | "maybe" type

function_decl     -> "pounce" IDENTIFIER "(" parameters? ")" return_type? block
parameters        -> parameter ("," parameter)*
parameter         -> IDENTIFIER type_annotation?
return_type       -> ":" type

if_statement      -> "sniff" expression block
                     ("peek" expression block)*
                     ("nap" block)?

hunt_statement    -> "hunt" IDENTIFIER "in" expression block
                   | "hunt" expression block

stash_statement   -> "stash" expression "," expression
purr_statement    -> "purr" expression
gift_statement    -> "gift" expression?
hiss_statement    -> "hiss" expression
leap_statement    -> "leap"
prowl_statement   -> "prowl"

block             -> "{" statement* "}"

expression        -> assignment
assignment        -> logical_or ("=" assignment)?
logical_or        -> logical_and ("orpaw" logical_and)*
logical_and       -> equality ("andpaw" equality)*
equality          -> comparison (("==" | "!=") comparison)*
comparison        -> range ((">" | ">=" | "<" | "<=") range)*
range             -> term (".." term)?
term              -> factor (("+" | "-") factor)*
factor            -> power (("*" | "/" | "%") power)*
power             -> unary ("**" power)?
unary             -> ("nah" | "-" | "+") unary
                   | postfix
postfix           -> primary (call | index)*
call              -> "(" arguments? ")"
index             -> "[" expression "]"
arguments         -> expression ("," expression)*

primary           -> PAW_LITERAL
                   | WHISKER_LITERAL
                   | MEOW_LITERAL
                   | "purring"
                   | "hissing"
                   | "emptybowl"
                   | IDENTIFIER
                   | list_literal
                   | listen_expression
                   | snatch_expression
                   | count_expression
                   | sniffout_expression
                   | conversion_expression
                   | "(" expression ")"

list_literal      -> "[" arguments? "]"
listen_expression -> "listen" expression?
snatch_expression -> "snatch" expression ("," expression)?
count_expression  -> "count" expression
sniffout_expression -> "sniffout" expression "," expression
conversion_expression -> ("paw" | "whisker" | "meow" | "mood") "(" expression ")"
```

### 19.1 Assignment targets

Only valid assignable targets may appear on the left side of `=`.

In v1 this includes:

- mutable identifiers
- indexed elements whose root binding is mutable

Invalid:

```nyarn
(2 + 2) = 5
```

### 19.2 Operator precedence

Highest to lowest:

1. calls and indexing
2. unary `nah`, unary `+`, unary `-`
3. exponentiation `**` (right-associative)
4. `*`, `/`, `%`
5. `+`, `-`
6. range `..`
7. `<`, `<=`, `>`, `>=`
8. `==`, `!=`
9. `andpaw`
10. `orpaw`
11. assignment `=` (right-associative)

---

## 20. Reference interpreter architecture

The Go implementation should be split into clear, testable units.

Recommended repository structure:

```text
cmd/
  nyarn/
    main.go

internal/
  lexer/
  parser/
  ast/
  checker/
  interpreter/
  runtime/
  diagnostic/

examples/
tests/
docs/
```

Responsibilities:

```text
lexer        source -> tokens with source spans
parser       tokens -> AST
ast          syntax tree node definitions
checker      names, types, narrowing, mutability, arity, return analysis
interpreter  tree-walk execution of checked AST
runtime      values, environments, functions, ranges, call stack
 diagnostic  structured Nyarn diagnostics and rendering
```

The checker and interpreter are intentionally separate.

This makes errors easier to catch before execution and keeps future editor/LSP tooling possible without reusing runtime evaluation as the type system.

Go's standard library should be preferred wherever practical. Nyarn v1 should avoid unnecessary dependencies.

---

## 21. Explicit v1 non-goals

Nyarn v1 intentionally does **not** include:

- modules/imports
- `pspsps` implementation
- records/structs/classes
- `breed` implementation
- nested functions
- closures
- default function arguments
- list slicing
- custom range steps
- integer-division operator
- catchable errors
- `carefully` / `hissback` implementation
- formatter / `nyarn fmt`
- object methods
- rich collection APIs such as map/filter/reduce/sort/reverse
- user-defined operators
- macros
- generics beyond the built-in `clowder T` / `maybe T` type forms

---

## 22. Canonical examples

### 22.1 Hello Nyarn

```nyarn
mew nyame = "Nyarn"
mew happy: mood = purring

sniff happy {
    purr "Henlo from {nyame}!"
}
nap {
    purr "zzz..."
}
```

### 22.2 Input and conversion

```nyarn
mew nyame = listen "Nyame: "
mew age_text = listen "Age: "
mew age: paw = paw(age_text)

purr "Henlo, {nyame}. You are {age}."
```

### 22.3 Mutable clowder

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]

stash cats, "Amora"
mew last = snatch cats

purr "removed: {last}"
purr "remaining cats: {count cats}"
```

### 22.4 Range and loop control

```nyarn
hunt i in 0..10 {
    sniff i == 3 {
        prowl
    }

    sniff i == 8 {
        leap
    }

    purr i
}
```

### 22.5 Recursion

```nyarn
pounce factorial(n: paw): paw {
    sniff n <= 1 {
        gift 1
    }

    gift n * factorial(n - 1)
}

purr factorial(5)
```

---

## 23. Public personality

The language specification is authoritative and should remain precise.

The public README, examples, and diagnostics are allowed to be much less dignified.

Canonical project attitude:

> Nyarn is what happens when someone gives a cat access to language design.

And:

> Is this necessary? No.
>
> Is that going to stop us? Also no.

The humor must never make a technical failure harder to understand.

---

## 24. Compatibility rule for v1

Until Nyarn v1 is formally released, this document is the source of truth for implementation work and may evolve through explicit design revisions.

Once a public v1 release is tagged, incompatible syntax or semantic changes should require a versioned language change rather than silently changing existing `.nyarn` programs.
