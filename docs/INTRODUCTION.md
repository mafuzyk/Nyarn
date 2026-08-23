# 🐈‍⬛ Nyarn v1 — Introduction

> **Nyarn is what happens when someone gives a cat access to language design.**
>
> This guide is the friendly way into Nyarn v1. It assumes you know basic programming ideas, but it does **not** assume you already understand Nyarn, its terminology, or why integers are called `paw`.

> [!IMPORTANT]
> Nyarn v1 is currently being implemented. The syntax and semantics in this guide describe the approved v1 language design. For normative behavior, see [`SPEC.md`](./SPEC.md). For the exhaustive user-facing reference, see [`NYARN_V1_GUIDE.md`](./NYARN_V1_GUIDE.md).

---

## 1. What is Nyarn?

Nyarn is a small interpreted programming language with:

- its own lexer and parser;
- its own AST;
- optional type annotations;
- a semantic checker;
- lexical scopes;
- functions and recursion;
- immutable-by-default bindings;
- homogeneous lists;
- a tree-walk interpreter written in Go;
- diagnostics that are useful first and unhinged second.

It is **not** Python with keywords renamed to cat noises.

The feline vocabulary is mostly cosmetic. The semantics are intentionally consistent and predictable.

If you remember only one design rule, remember this:

> Nyarn is allowed to be silly. Nyarn is not allowed to be vague.

---

## 2. Your first Nyarn program

Nyarn files use the `.nyarn` extension.

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

Read it like this:

1. `mew nyame = "Nyarn"` creates an immutable variable named `nyame`.
2. `mew happy: mood = purring` creates an immutable boolean-like value.
3. `sniff happy` is Nyarn's `if`.
4. `purr` prints one expression and adds a newline.
5. `nap` is Nyarn's `else`.

The same idea in boring-language vocabulary would be roughly:

```text
let nyame = "Nyarn"
let happy: bool = true

if happy {
    print("Henlo from Nyarn!")
} else {
    print("zzz...")
}
```

Nyarn simply refuses to use several perfectly normal words.

---

## 3. The vocabulary you actually need

You do **not** need to memorize the entire language before writing anything. Start with these:

| Nyarn | Meaning |
| --- | --- |
| `mew` | declare an immutable binding |
| `scratch` | declare a mutable binding |
| `purr` | print |
| `listen` | read input |
| `sniff` | if |
| `peek` | else if |
| `nap` | else |
| `hunt` | loop |
| `leap` | break |
| `prowl` | continue |
| `pounce` | function declaration |
| `gift` | return |
| `hiss` | raise a runtime error |

And the core types:

| Nyarn | Meaning |
| --- | --- |
| `paw` | integer |
| `whisker` | floating-point number |
| `meow` | string |
| `mood` | boolean |
| `clowder T` | homogeneous list of `T` |
| `maybe T` | optional `T` |

Special values:

```nyarn
purring    # true
hissing    # false
emptybowl  # absence / null
```

Yes, `emptybowl` is null.

The tragedy is part of the type system.

---

## 4. Variables: `mew` and `scratch`

Nyarn defaults to immutability.

```nyarn
mew nyame = "Mafu"
```

Once declared with `mew`, the binding cannot be reassigned:

```nyarn
mew nyame = "Mafu"
nyame = "Nyarn"  # HISS
```

When you actually want mutation, use `scratch`:

```nyarn
scratch lives = 9
lives = lives - 1
```

The distinction is intentional:

```text
mew      -> this should stay stable
scratch  -> yes, I intend to mutate this
```

For lists, the rule also protects mutation through an immutable binding:

```nyarn
mew cats = ["Milo", "Luna"]
cats[0] = "Dexter"  # HISS
```

But this is valid:

```nyarn
scratch cats = ["Milo", "Luna"]
cats[0] = "Dexter"
```

> [!TIP]
> If you are unsure whether something needs to change, start with `mew`. Make mutation explicit only when you need it.

---

## 5. Types without suffering

Type annotations are optional.

```nyarn
mew age = 17
mew height = 1.72
mew nyame = "Mafu"
mew sleepy = purring
```

Nyarn infers:

```text
17         -> paw
1.72       -> whisker
"Mafu"     -> meow
purring    -> mood
```

You can also annotate explicitly:

```nyarn
mew age: paw = 17
mew height: whisker = 1.72
mew nyame: meow = "Mafu"
mew sleepy: mood = purring
```

An explicit annotation is a contract:

```nyarn
mew age: paw = "seventeen"
```

Nyarn rejects this before normal execution.

The cat is silly, not permissive.

---

## 6. `mood`: booleans without `true` or `false`

Nyarn's boolean type is `mood`.

```nyarn
mew happy: mood = purring
mew angry: mood = hissing
```

Nyarn has **no general truthiness**.

This is valid:

```nyarn
mew hungry = purring

sniff hungry {
    purr "FOOD"
}
```

This is not:

```nyarn
mew lives = 9

sniff lives {
    purr "???"
}
```

A condition must be a `mood`.

Logical operators are:

```text
andpaw  -> and
orpaw   -> or
nah     -> not
```

Example:

```nyarn
sniff hungry andpaw nah sleepy {
    purr "time to destroy the kitchen"
}
```

---

## 7. Conditions: `sniff`, `peek`, `nap`

Nyarn conditional chains look like this:

```nyarn
sniff hungry {
    purr "FOOD"
}
peek sleepy {
    purr "zzz"
}
nap {
    purr "mrrp"
}
```

Translation:

```text
sniff -> if
peek  -> else if
nap   -> else
```

There may be multiple `peek` branches and at most one final `nap`.

Every branch creates its own lexical scope.

```nyarn
mew nyame = "Mafu"

sniff purring {
    mew nyame = "Nyarn"
    purr nyame  # Nyarn
}

purr nyame      # Mafu
```

Shadowing between nested scopes is allowed. Redeclaring the same name in the same scope is not.

---

## 8. Loops: hunt things

Nyarn uses `hunt` for both while-like and for-like loops.

### Conditional hunt

```nyarn
scratch lives = 3

hunt lives > 0 {
    purr "{lives} lives left"
    lives = lives - 1
}
```

### Iteration hunt

```nyarn
mew cats = ["Milo", "Luna", "Dexter"]

hunt cat in cats {
    purr cat
}
```

Loop control:

```nyarn
leap   # break
prowl  # continue
```

Example:

```nyarn
hunt cat in cats {
    sniff cat == "Luna" {
        prowl
    }

    sniff cat == "Dexter" {
        leap
    }

    purr cat
}
```

The `hunt` iteration variable is immutable.

---

## 9. Ranges

Nyarn v1 supports half-open integer ranges.

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

So:

```text
0..10
```

means 0 through 9.

Range bounds are `paw` values in v1. Custom steps are intentionally not part of v1.

---

## 10. Functions: `pounce` and `gift`

Functions are declared with `pounce`.

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}

purr add(2, 3)
```

`gift` is return.

Parameter and return annotations are optional:

```nyarn
pounce greet(nyame) {
    purr "Henlo, {nyame}"
}
```

A function may simply end without `gift`. In that case it returns `emptybowl`.

This is also allowed:

```nyarn
pounce stop() {
    gift
}
```

which also returns `emptybowl`.

Nyarn v1 functions:

- are declared only at top level;
- support recursion;
- are hoisted;
- require the exact declared number of arguments;
- do not support default parameters;
- do not support nested functions or closures.

Because functions are hoisted, this is valid:

```nyarn
greet("Mafu")

pounce greet(nyame: meow) {
    purr "Henlo, {nyame}"
}
```

Variables are **not** hoisted.

---

## 11. Clowders: lists, except with more cats

A group of cats is called a clowder, therefore:

```nyarn
mew cats: clowder meow = ["Milo", "Luna", "Dexter"]
```

Nyarn lists are homogeneous.

```nyarn
mew numbers = [1, 2, 3]
```

is inferred as `clowder paw`.

This is invalid:

```nyarn
mew chaos = [1, "cat", purring]
```

A bare empty clowder needs a type annotation:

```nyarn
mew cats: clowder meow = []
```

Indexing is familiar:

```nyarn
purr cats[0]
purr cats[-1]
```

Negative indices work Python-style.

Out-of-range access does not return `emptybowl`. It HISSes.

Core clowder operations:

```nyarn
stash cats, "Amora"       # append
mew last = snatch cats     # remove final item
mew gone = snatch cats, 1  # remove index 1
purr count cats            # length
purr sniffout cats, "Milo" # membership
```

Mutating operations require a `scratch` binding.

---

## 12. Strings and interpolation

Nyarn v1 uses double-quoted strings.

```nyarn
mew nyame = "Mafu"
```

Supported escapes include:

```text
\n
\t
\"
\\
```

String interpolation uses braces:

```nyarn
mew lives = 9
purr "{nyame} has {lives} lives"
purr "next life: {lives - 1}"
```

Interpolation accepts expressions, not only identifiers.

Literal braces are doubled:

```nyarn
purr "{{hello}}"
```

prints:

```text
{hello}
```

Nyarn does **not** coerce arbitrary values into strings with `+`:

```nyarn
purr "lives: " + 9  # HISS
```

Use interpolation:

```nyarn
purr "lives: {9}"
```

or explicit conversion:

```nyarn
purr "lives: " + meow(9)
```

---

## 13. Input and conversion

Read text using `listen`:

```nyarn
mew nyame = listen "Nyame: "
purr "Henlo, {nyame}"
```

Or without a prompt:

```nyarn
mew input = listen
```

`listen` returns `meow`.

To convert values, call the type name:

```nyarn
mew age_text = listen "Age: "
mew age: paw = paw(age_text)
```

Common conversions:

```nyarn
paw("17")
whisker("1.72")
meow(42)
mood("purring")
```

Conversions are explicit because Nyarn refuses to become JavaScript by accident.

---

## 14. Optional values: `maybe` and `emptybowl`

A normal type cannot contain `emptybowl`.

Invalid:

```nyarn
mew nickname: meow = emptybowl
```

Use `maybe`:

```nyarn
mew nickname: maybe meow = emptybowl
```

Nyarn narrows the type when you check against `emptybowl`:

```nyarn
sniff nickname != emptybowl {
    # nickname is meow in this branch
    purr "Henlo, {nickname}"
}
```

The inverse works too:

```nyarn
sniff nickname == emptybowl {
    purr "No nickname. Tragic."
}
nap {
    # nickname is meow here
    purr nickname
}
```

This gives Nyarn optional values without requiring unsafe implicit nullability everywhere.

---

## 15. Numbers

Arithmetic operators:

```text
+  -  *  /  %  **
```

Comparison operators:

```text
==  !=  >  >=  <  <=
```

Nyarn has one implicit numeric promotion:

```text
paw -> whisker
```

So:

```nyarn
mew a: paw = 5
mew b: whisker = 2.5
purr a + b
```

produces a `whisker` value.

Division always returns `whisker`:

```nyarn
purr 5 / 2
# 2.5
```

Exponentiation is right-associative:

```nyarn
2 ** 3 ** 2
```

means:

```text
2 ** (3 ** 2)
```

Numeric comparisons permit `paw -> whisker` promotion, but unrelated types are not magically coerced.

```nyarn
5 == 5.0   # purring
5 == "5"   # HISS
```

---

## 16. Raising an error with `hiss`

Use `hiss` to explicitly fail:

```nyarn
sniff lives < 0 {
    hiss "cats cannot have negative lives"
}
```

In v1, a `hiss` cannot be caught. It propagates to the top level and ends file execution with a nonzero status.

Future versions reserve `carefully` and `hissback` for error-catching semantics, but they are **not** implemented in v1.

---

## 17. Diagnostics: useful first, unhinged second

Nyarn's diagnostics are supposed to help you before making jokes.

Example:

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

8 | sniff lives {
          ^^^^^

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

The useful part tells you:

- what kind of failure happened;
- where it happened;
- which source span caused it;
- what was expected;
- sometimes how to fix it.

The cat panic comes after.

Runtime errors are designed to include Nyarn-level stack traces when available.

Raw Go panic output should not leak into normal Nyarn errors.

---

## 18. Comments and formatting

Single-line comments use `#`:

```nyarn
# entire line
mew nyame = "Mafu"  # inline
```

Nyarn v1 has no multiline comments.

Semicolons are not used.

Statements normally end at newlines, while line breaks are allowed freely inside grouping constructs:

```nyarn
mew result = (
    10 +
    20 +
    30
)
```

---

## 19. A small complete program

Here is enough Nyarn to tie most of the introduction together:

```nyarn
pounce greet(nyame: meow, lives: paw) {
    purr "Henlo, {nyame}."
    purr "You currently have {lives} lives."
}

mew nyame = listen "Nyame: "
scratch lives: paw = 3
scratch cats: clowder meow = [nyame]

stash cats, "Nyarn"
stash cats, "Milo"

greet(nyame, lives)

hunt cat in cats {
    sniff cat == "Milo" {
        prowl
    }

    purr "cat detected: {cat}"
}

hunt lives > 0 {
    purr "life {lives}"
    lives = lives - 1
}

sniff lives == 0 {
    purr "empty life inventory"
}
nap {
    hiss "physics has failed"
}
```

If you can read that without constantly consulting this page, congratulations: you already understand most of Nyarn v1's everyday syntax.

---

## 20. What Nyarn v1 deliberately does not have

Nyarn v1 is small on purpose.

It does not include:

- imports/modules;
- `pspsps` implementation;
- classes;
- records/structs;
- `breed` implementation;
- nested functions;
- closures;
- default arguments;
- slicing;
- custom range steps;
- integer-division syntax;
- catchable errors;
- `carefully` / `hissback` implementation;
- formatter;
- LSP;
- package manager;
- bytecode VM;
- JIT.

A language does not become better because its first version contains every feature its creator has ever heard of.

Sometimes the cat should simply stop adding things.

---

## 21. Where to go next

You now have three useful documents:

1. **This introduction** — learn Nyarn by example.
2. **[`NYARN_V1_GUIDE.md`](./NYARN_V1_GUIDE.md)** — exhaustive user-facing guide and reference.
3. **[`SPEC.md`](./SPEC.md)** — normative language specification and source of truth.

If the guide and the specification ever disagree, the specification wins.

The spec wins over vibes.

Usually.

---

## Quick cheat sheet

```text
mew       immutable declaration
scratch   mutable declaration

paw       integer
whisker   float
meow      string
mood      boolean
clowder   homogeneous list
maybe     optional type

purring   true
hissing   false
emptybowl null/absence

sniff     if
peek      else if
nap       else

andpaw    and
orpaw     or
nah       not

hunt      loop
leap      break
prowl     continue

pounce    function
gift      return

purr      print
listen    input
hiss      raise runtime error

stash     append
snatch    remove + return
count     length
sniffout  membership / substring
```

> **Welcome to Nyarn.**
>
> Write code. Chase bugs. Hiss at type errors.
