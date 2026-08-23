<div align="center">

# 🐈‍⬛ Nyarn

### A programming language for cats, humans, and other creatures with questionable judgment.

**Nyarn is what happens when someone gives a cat access to language design.**

> Is this necessary? **No.**  
> Is that going to stop us? **Also no.**

</div>

---

## So... what is Nyarn?

Nyarn is a small interpreted programming language with its own lexer, parser, AST, semantic checker, runtime, diagnostics, and tree-walk interpreter — all planned to be implemented in Go.

It is also a language where:

- integers are called `paw`;
- floats are called `whisker`;
- strings are called `meow`;
- booleans are called `mood`;
- `purring` means true;
- `hissing` means false;
- `emptybowl` means null;
- `purr` prints things;
- `hiss` ruins your day;
- and `nyame` is not a keyword, but using `name` instead is spiritually questionable.

Nyarn is a joke.

Nyarn is also an actual programming language project.

This has created several problems.

> [!IMPORTANT]
> **Nyarn v1 is currently being implemented.** The language design and specification are approved, but the reference interpreter is not finished yet. Code examples in this README describe the intended v1 language unless stated otherwise.

---

## Behold: programming

```nyarn
mew nyame = "Nyarn"
scratch lives: paw = 9
mew happy: mood = purring

sniff happy andpaw lives > 0 {
    purr "Henlo from {nyame}!"
    purr "The cat has {lives} lives. Probably."
}

nap {
    hiss "HOW DID THE CAT RUN OUT OF LIVES???"
}
```

Perfectly normal programming language. Nothing to see here.

---

## Is Nyarn just Python with cat words?

No.

That would have been significantly easier.

Nyarn has its own language design and reference implementation architecture:

```text
source (.nyarn)
      ↓
    lexer
      ↓
    tokens
      ↓
    parser
      ↓
      AST
      ↓
semantic checker
      ↓
tree-walk interpreter
      ↓
 runtime + call stack
```

The checker and interpreter are intentionally separate, because even cats deserve static diagnostics before runtime chaos begins.

The reference implementation is being written in **Go**, with the standard library preferred wherever practical.

---

## A deeply serious type system

| Nyarn | Boring people call it | Example |
| --- | --- | --- |
| `paw` | integer | `mew lives: paw = 9` |
| `whisker` | float | `mew height: whisker = 1.72` |
| `meow` | string | `mew nyame: meow = "Mafu"` |
| `mood` | boolean | `mew happy: mood = purring` |
| `clowder T` | homogeneous list | `mew cats: clowder meow = ["Milo", "Luna"]` |
| `maybe T` | optional / nullable value | `mew food: maybe meow = emptybowl` |

Special values:

```nyarn
purring    # true
hissing    # false
emptybowl  # null / absence / tragedy
```

There is no general truthiness.

```nyarn
mew lives = 9

sniff lives {
    purr "surely this is fine"
}
```

It is not fine.

Nyarn expects a `mood`. The cat will not guess what you meant.

---

## `mew` and `scratch`

Nyarn defaults to immutability.

```nyarn
mew nyame = "Mafu"
```

`mew` creates an immutable binding.

```nyarn
nyame = "Definitely Not Mafu"
# HISS
```

If you actually intend to mutate something, admit your crimes explicitly:

```nyarn
scratch lives: paw = 9
lives = 8
```

This also applies to clowders:

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]
cats[0] = "Dexter"
```

With `mew`, the cat has been told not to scratch it.

---

## Control flow, but feline

Why write `if / else if / else` when you can instead inspect reality like a suspicious animal?

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

Translation for non-cats:

```text
sniff  -> if
peek   -> else if
nap    -> else
```

Logical operators are also completely reasonable:

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

## Hunt things

Nyarn uses `hunt` for both conditional and iteration loops.

```nyarn
hunt lives > 0 {
    lives = lives - 1
}
```

```nyarn
hunt cat in cats {
    purr cat
}
```

Loop control:

```text
leap   -> break
prowl  -> continue
```

Ranges are half-open:

```nyarn
hunt i in 0..3 {
    purr i
}
```

prints:

```text
0
1
2
```

The cat respects off-by-one errors enough to define this explicitly.

---

## Functions are called `pounce`

Obviously.

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}

purr add(2, 3)
```

`gift` means return because the function has brought you something.

Function parameter and return annotations are optional:

```nyarn
pounce greet(nyame) {
    purr "Henlo, {nyame}"
}
```

Nyarn v1 supports recursion and top-level function hoisting, but intentionally does **not** support closures or nested functions yet.

We are committing crimes responsibly.

---

## Clowders

A group of cats is called a clowder.

Therefore a Nyarn list is called a `clowder`.

I do not make the rules.

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]

stash cats, "Amora"
mew last = snatch cats

purr "removed: {last}"
purr "cats remaining: {count cats}"
```

Core v1 collection operations:

```text
stash     append an item
snatch    remove and return an item
count     get string/list length
sniffout  membership / substring check
```

Negative indexing works because Python users have suffered enough:

```nyarn
purr cats[-1]
```

Out-of-bounds access does **not** silently return `emptybowl`.

It HISSes.

---

## Input, output, and existential dread

Output:

```nyarn
purr "mrrp"
```

Input:

```nyarn
mew nyame = listen "Nyame: "
purr "Henlo, {nyame}"
```

Explicit conversions use the type names:

```nyarn
mew age_text = listen "Age: "
mew age: paw = paw(age_text)
```

No, Nyarn will not silently decide that `"5" + 2` should be `7`.

We have standards.

Low standards, perhaps, but standards.

---

## Errors: useful first, unhinged second

Nyarn diagnostics have one sacred design rule:

> **Useful first. Unhinged second.**

A real error should tell you what happened, where it happened, and ideally how to fix it.

Then the cat is allowed to lose its mind.

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

8 | sniff lives {
          ^^^^^

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

Or perhaps:

```text
HISS [runtime]
maximum call depth exceeded

THE YARN IS TOO TANGLED!!!
IS THE PROGRAM RECURSING?!
CAN'T KEEP UP!!!
```

Runtime failures are intended to include Nyarn-level stack traces.

Raw Go panic dumps are not a personality trait and should not leak into normal user-facing errors.

---

## Planned CLI

Once the reference interpreter lands, both of these forms are specified:

```bash
nyarn hello.nyarn
nyarn run hello.nyarn
```

Running Nyarn without a file launches the REPL:

```text
$ nyarn
Nyarn REPL
> mew x = 9
> purr x
9
>
```

The REPL is designed to preserve state between entries and survive user errors without becoming permanently haunted.

---

## Current status

Nyarn is in active early development.

Already done:

- [x] Decide that integers should be called paws
- [x] Question absolutely none of our decisions
- [x] Design Nyarn v1
- [x] Write the language specification
- [x] Define the reference interpreter architecture
- [x] Write a TDD implementation plan
- [x] Accidentally become emotionally invested in a cat language

Reference interpreter:

- [ ] Lexer
- [ ] Diagnostics infrastructure
- [ ] AST
- [ ] Parser
- [ ] Type system
- [ ] Semantic checker
- [ ] Runtime
- [ ] Tree-walk interpreter
- [ ] Core built-ins
- [ ] File runner
- [ ] REPL
- [ ] CLI
- [ ] Conformance suite
- [ ] Release something another human can actually run
- [ ] Regret

The implementation plan is intentionally test-driven. We would like the cat to fail deterministically.

---

## Read the serious documents

Yes, there are serious documents. Somehow.

- **[Nyarn v1 Language Specification](docs/SPEC.md)** — the source of truth for syntax and semantics.
- **[Nyarn v1 Reference Interpreter Implementation Plan](docs/superpowers/plans/2026-08-23-nyarn-v1-reference-interpreter.md)** — the TDD roadmap for the Go implementation.

The spec wins over vibes.

Usually.

---

## Things v1 deliberately does NOT have

Before anyone opens an issue asking for inheritance:

- modules/imports;
- `pspsps` implementation;
- classes;
- structs/records;
- `breed` implementation;
- nested functions;
- closures;
- default arguments;
- slicing;
- custom range steps;
- integer division syntax;
- catchable errors;
- `carefully` / `hissback`;
- formatter;
- LSP;
- package manager;
- bytecode VM;
- JIT;
- enterprise blockchain AI cat microservices.

Some of these may come later.

One of them absolutely will not.

---

## Reserved future nonsense

The following words are intentionally reserved for future Nyarn versions:

```text
pspsps     imports / modules
breed      records / struct-like types
carefully  future error-catching construct
hissback   future error handler
```

Yes, imports are planned to be called `pspsps`.

This decision has survived multiple opportunities to stop it.

---

## Contributing

Nyarn is being built as a real language project, so contributions should respect the specification even when the specification says something profoundly unserious.

If you want to contribute:

1. read [`docs/SPEC.md`](docs/SPEC.md);
2. do not invent new v1 semantics in the middle of an implementation PR;
3. add tests for behavior changes;
4. keep diagnostics useful before adding cat panic;
5. avoid unnecessary dependencies;
6. please do not add object-oriented inheritance as your first contribution;
7. remember that `nyame` is culture, not syntax.

Bug reports, implementation fixes, test cases, diagnostic improvements, and tasteful feline nonsense are welcome.

---

## Design philosophy

Nyarn tries to live in a very specific and deeply unnecessary intersection:

- small enough to understand;
- weird enough to remember;
- strict enough to teach good language design;
- simple enough to implement without summoning LLVM on day one;
- unserious enough that an out-of-bounds list access can yell about yarn;
- serious enough that the out-of-bounds list access still points at the correct source span.

The humor is part of the language's identity.

Undefined behavior is not.

---

## License

Nyarn v1 is designed to be released under the **MIT License**.

The repository is still being bootstrapped, so check the repository files for the current license text before redistributing a snapshot.

---

<div align="center">

### 🧶 Write code. Chase bugs. Hiss at type errors.

**Nyarn** — because apparently `int`, `bool`, and `return` were too normal.

</div>
