# Nyarn v1 Reference Interpreter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Nyarn v1 reference implementation as a Go tree-walk interpreter that conforms to `docs/SPEC.md`, including lexer, parser, AST, semantic checker, runtime, interpreter, diagnostics, CLI, REPL, examples, and conformance tests.

**Architecture:** Nyarn uses a strict pipeline: source → lexer → tokens → parser → AST → semantic checker → tree-walk interpreter → runtime values/environments/call stack. Syntax and semantic checking are separated from evaluation so user programs fail before execution when an error can be known statically. The implementation stays dependency-light and uses focused internal packages with explicit interfaces between stages.

**Tech Stack:** Go; Go standard library; `testing` package; no third-party runtime dependencies unless a later explicit design revision approves one.

**Spec:** `docs/SPEC.md`

## Global Constraints

- Source extension is exactly `.nyarn`.
- Implementation language is Go.
- Execution model is a tree-walk interpreter.
- Normal file execution must not begin until lexing, parsing, and semantic checking succeed.
- Statements are newline-terminated; semicolons are not part of v1 syntax.
- Comments use `#`; v1 has no multiline comment syntax.
- `mew` bindings are immutable; `scratch` bindings are mutable.
- Only `mood` may be used as a condition; Nyarn has no general truthiness.
- The only implicit numeric promotion is `paw -> whisker`.
- `/` always returns `whisker`.
- `clowder T` is homogeneous; bare `[]` requires an explicit type.
- `emptybowl` is assignable only to `maybe T`.
- Functions are top-level only, support hoisting and recursion, and do not support closures.
- Nyarn v1 is single-file: no imports or module system.
- `pspsps`, `breed`, `carefully`, and `hissback` are reserved but not implemented.
- Core built-ins are `purr`, `listen`, conversions `paw/whisker/meow/mood`, `stash`, `snatch`, `count`, and `sniffout`.
- Diagnostics must be technically useful first and may add an unserious cat message second.
- User-caused Nyarn failures must never leak raw Go panics.
- CLI forms `nyarn file.nyarn` and `nyarn run file.nyarn` are equivalent.
- Running `nyarn` with no file starts a persistent REPL.
- Exit codes are: `0` success, `1` runtime/explicit `hiss`, `2` syntax/parser, `3` checker/type, `4` CLI/file/I/O.
- Prefer Go's standard library and avoid unnecessary dependencies.
- v1 explicitly excludes modules, records/classes, nested functions, closures, default arguments, slicing, custom range steps, integer division, catchable errors, formatter support, object methods, rich collection APIs, macros, and user-defined operators.

---

## File Structure

Create the implementation with these responsibilities:

```text
go.mod
cmd/
  nyarn/
    main.go
internal/
  source/
    span.go
  token/
    token.go
    kind.go
  lexer/
    lexer.go
    lexer_test.go
  ast/
    expr.go
    stmt.go
    type.go
  parser/
    parser.go
    expression.go
    statement.go
    parser_test.go
  types/
    type.go
    type_test.go
  checker/
    checker.go
    scope.go
    expression.go
    statement.go
    checker_test.go
  runtime/
    value.go
    environment.go
    function.go
    range.go
    signal.go
    runtime_test.go
  diagnostic/
    diagnostic.go
    render.go
    suggest.go
    diagnostic_test.go
  interpreter/
    interpreter.go
    expression.go
    statement.go
    builtin.go
    interpreter_test.go
  runner/
    runner.go
    runner_test.go
  repl/
    repl.go
    repl_test.go
examples/
  hello.nyarn
  loops.nyarn
  functions.nyarn
  clowder.nyarn
tests/
  conformance_test.go
README.md
LICENSE
```

The package dependency direction must remain one-way:

```text
source
  ↓
token
  ↓
lexer
  ↓
ast
  ↓
parser
  ↓
types
  ↓
checker
  ↓
runtime
  ↓
interpreter
  ↓
runner / repl / cmd
```

`diagnostic` may depend on `source`, but core parsing/checking/runtime packages must not depend on CLI or REPL packages.

---

### Task 1: Bootstrap the Go module, source spans, tokens, and the v1 lexer

**Files:**
- Create: `go.mod`
- Create: `internal/source/span.go`
- Create: `internal/token/kind.go`
- Create: `internal/token/token.go`
- Create: `internal/lexer/lexer.go`
- Create: `internal/lexer/lexer_test.go`

**Interfaces:**
- Produces:
  - `source.Position{Offset int, Line int, Column int}`
  - `source.Span{Start Position, End Position}`
  - `source.Join(a, b Span) Span`
  - `token.Token{Kind Kind, Lexeme string, Literal any, Span source.Span}`
  - `lexer.Lex(filename, input string) ([]token.Token, []diagnostic.Diagnostic)` once diagnostics exist; until Task 2, use `([]token.Token, error)` and convert in Task 2.
- Recognizes every v1 keyword and reserved future word from `docs/SPEC.md`.
- Emits explicit `NEWLINE` and terminal `EOF` tokens.
- Ignores spaces/tabs outside strings and ignores `#` comments through end-of-line.

- [ ] **Step 1: Create the module and failing lexer tests**

`go.mod`:

```go
module github.com/mafuzyk/Nyarn

go 1.24
```

`internal/lexer/lexer_test.go` must include table-driven tests for keywords, identifiers, numeric literals, strings, operators, punctuation, comments, Unicode identifiers, newlines, and the `0..10` range-vs-float boundary. Define the shared helper in that file:

```go
func assertKinds(t *testing.T, got []token.Token, want []token.Kind) {
    t.Helper()
    if len(got) != len(want) {
        t.Fatalf("token count = %d, want %d", len(got), len(want))
    }
    for i := range want {
        if got[i].Kind != want[i] {
            t.Fatalf("token %d kind = %v, want %v", i, got[i].Kind, want[i])
        }
    }
}

func TestLexCanonicalDeclaration(t *testing.T) {
    got, err := Lex("test.nyarn", `mew nyame: meow = "Mafu"` + "\n")
    if err != nil {
        t.Fatal(err)
    }

    kinds := []token.Kind{
        token.MEW, token.IDENTIFIER, token.COLON, token.TYPE_MEOW,
        token.EQUAL, token.STRING, token.NEWLINE, token.EOF,
    }
    assertKinds(t, got, kinds)
}

func TestLexUnicodeIdentifier(t *testing.T) {
    got, err := Lex("test.nyarn", "mew coração = purring\n")
    if err != nil {
        t.Fatal(err)
    }
    if got[1].Lexeme != "coração" {
        t.Fatalf("identifier = %q", got[1].Lexeme)
    }
}

func TestCommentsStopAtNewline(t *testing.T) {
    got, err := Lex("test.nyarn", "mew x = 1 # cat noise\npurr x\n")
    if err != nil {
        t.Fatal(err)
    }
    assertKinds(t, got, []token.Kind{
        token.MEW, token.IDENTIFIER, token.EQUAL, token.PAW,
        token.NEWLINE, token.PURR, token.IDENTIFIER, token.NEWLINE, token.EOF,
    })
}

func TestLexRangeDoesNotBecomeFloat(t *testing.T) {
    got, err := Lex("test.nyarn", "hunt i in 0..10 {}\n")
    if err != nil {
        t.Fatal(err)
    }
    assertKinds(t, got, []token.Kind{
        token.HUNT, token.IDENTIFIER, token.IN, token.PAW, token.DOT_DOT,
        token.PAW, token.LEFT_BRACE, token.RIGHT_BRACE, token.NEWLINE, token.EOF,
    })
}
```

- [ ] **Step 2: Run the lexer tests and confirm failure**

Run:

```bash
go test ./internal/lexer -run TestLex -v
```

Expected: compilation failure because the lexer/token/source packages do not yet exist.

- [ ] **Step 3: Implement positions, spans, token kinds, keyword lookup, and scanning**

`internal/token/kind.go` must define explicit constants for literals, punctuation, operators, newline/EOF, all v1 keywords, and reserved words. Include a keyword map:

```go
var Keywords = map[string]Kind{
    "mew": MEW, "scratch": SCRATCH,
    "sniff": SNIFF, "peek": PEEK, "nap": NAP,
    "hunt": HUNT, "in": IN, "leap": LEAP, "prowl": PROWL,
    "pounce": POUNCE, "gift": GIFT, "purr": PURR, "hiss": HISS,
    "listen": LISTEN, "andpaw": ANDPAW, "orpaw": ORPAW, "nah": NAH,
    "paw": TYPE_PAW, "whisker": TYPE_WHISKER, "meow": TYPE_MEOW,
    "mood": TYPE_MOOD, "clowder": CLOWDER, "maybe": MAYBE,
    "purring": PURRING, "hissing": HISSING, "emptybowl": EMPTYBOWL,
    "stash": STASH, "snatch": SNATCH, "count": COUNT, "sniffout": SNIFFOUT,
    "pspsps": RESERVED_PSPSPS, "breed": RESERVED_BREED,
    "carefully": RESERVED_CAREFULLY, "hissback": RESERVED_HISSBACK,
}
```

Scanner requirements:
- UTF-8 identifiers use Unicode letters plus `_`; digits are allowed after the first rune.
- `paw` literal: decimal integer.
- `whisker` literal: decimal numeric literal containing a decimal point.
- Strings support `\n`, `\t`, `\"`, `\\`.
- `{{` and `}}` are retained in string content for interpolation handling later.
- Recognize `== != >= <= ** ..`.
- A lone unsupported character returns a lexer error with its source span.

- [ ] **Step 4: Run lexer tests and full package tests**

Run:

```bash
go test ./internal/lexer ./internal/token ./internal/source -v
```

Expected: PASS.

- [ ] **Step 5: Commit the lexer foundation**

```bash
git add go.mod internal/source internal/token internal/lexer
git commit -m "feat: add Nyarn lexer foundation"
```

---

### Task 2: Add structured diagnostics and replace lexer errors with Nyarn diagnostics

**Files:**
- Create: `internal/diagnostic/diagnostic.go`
- Create: `internal/diagnostic/render.go`
- Create: `internal/diagnostic/suggest.go`
- Create: `internal/diagnostic/diagnostic_test.go`
- Modify: `internal/lexer/lexer.go`
- Modify: `internal/lexer/lexer_test.go`

**Interfaces:**
- Produces:
  - `diagnostic.Kind` with `Syntax`, `Type`, `Runtime`, `Name`, `IO`, `Internal`
  - `diagnostic.Diagnostic{Kind, Filename, Span, Message, Hint, CatMessage, Frames}`
  - `diagnostic.Render(w io.Writer, sourceText string, d Diagnostic)`
  - `diagnostic.Suggest(input string, candidates []string) (string, bool)`
- Changes lexer signature to:
  - `func Lex(filename, input string) ([]token.Token, []diagnostic.Diagnostic)`

- [ ] **Step 1: Write failing diagnostic rendering and suggestion tests**

Required tests:

```go
func TestRenderTypeDiagnostic(t *testing.T) {
    d := Diagnostic{
        Kind: Type,
        Filename: "main.nyarn",
        Span: source.Span{
            Start: source.Position{Line: 8, Column: 7},
            End: source.Position{Line: 8, Column: 12},
        },
        Message: "expected mood, got paw",
        CatMessage: "THAT IS NOT A MOOD!!!",
    }

    var buf bytes.Buffer
    Render(&buf, "sniff lives {\n", d)
    out := buf.String()

    for _, want := range []string{
        "HISS [type]", "main.nyarn:8:7",
        "expected mood, got paw", "THAT IS NOT A MOOD!!!",
    } {
        if !strings.Contains(out, want) {
            t.Fatalf("missing %q in %q", want, out)
        }
    }
}

func TestSuggestNearbyIdentifier(t *testing.T) {
    got, ok := Suggest("nymae", []string{"nyame", "lives", "cats"})
    if !ok || got != "nyame" {
        t.Fatalf("got %q, %v", got, ok)
    }
}
```

- [ ] **Step 2: Run diagnostics tests and verify they fail**

```bash
go test ./internal/diagnostic -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement structured diagnostics and source excerpt rendering**

Requirements:
- Render first line as `HISS [kind] at file:line:column` when a location exists.
- Render the primary message before the cat message.
- Render a source line and caret span when source text is available.
- Render `did you mean ...?` only when `Hint` is non-empty.
- Implement suggestion with a small Levenshtein distance threshold proportional to candidate length; ties choose the lexicographically first candidate so tests are deterministic.
- `Internal` errors render `NYARN INTERNAL ERROR`, not `HISS [internal]`.

- [ ] **Step 4: Convert lexer failures to diagnostics and run tests**

Update lexer tests to expect diagnostics:

```go
tokens, diags := Lex("bad.nyarn", "mew x = @\n")
if len(diags) != 1 || diags[0].Kind != diagnostic.Syntax {
    t.Fatalf("diagnostics = %#v", diags)
}
```

Run:

```bash
go test ./internal/diagnostic ./internal/lexer -v
```

Expected: PASS.

- [ ] **Step 5: Commit diagnostics**

```bash
git add internal/diagnostic internal/lexer
git commit -m "feat: add structured Nyarn diagnostics"
```

---

### Task 3: Define the complete AST and type-syntax nodes

**Files:**
- Create: `internal/ast/expr.go`
- Create: `internal/ast/stmt.go`
- Create: `internal/ast/type.go`
- Create: `internal/ast/ast_test.go`

**Interfaces:**
- Produces:
  - `ast.Program{Statements []Stmt, NodeSpan source.Span}`
  - `ast.Expr` and concrete expression structs
  - `ast.Stmt` and concrete statement structs
  - `ast.TypeExpr` for `paw`, `whisker`, `meow`, `mood`, `clowder T`, `maybe T`
- Every AST node must expose `Span() source.Span`.
- The parser and checker must not depend on concrete runtime values.

Minimum expression nodes:
`PawLiteral`, `WhiskerLiteral`, `MeowLiteral`, `MoodLiteral`, `EmptyBowlLiteral`, `IdentifierExpr`, `ListExpr`, `UnaryExpr`, `BinaryExpr`, `AssignmentExpr`, `CallExpr`, `IndexExpr`, `RangeExpr`, `ListenExpr`, `SnatchExpr`, `CountExpr`, `SniffoutExpr`, `ConversionExpr`, `InterpolatedStringExpr`.

Minimum statement nodes:
`MewDecl`, `ScratchDecl`, `ExprStmt`, `PurrStmt`, `StashStmt`, `GiftStmt`, `HissStmt`, `LeapStmt`, `ProwlStmt`, `BlockStmt`, `IfStmt`, `HuntWhileStmt`, `HuntInStmt`, `PounceDecl`.

- [ ] **Step 1: Write failing AST span tests**

Define the test helper explicitly:

```go
func span(sl, sc, el, ec int) source.Span {
    return source.Span{
        Start: source.Position{Line: sl, Column: sc},
        End:   source.Position{Line: el, Column: ec},
    }
}

func TestBinaryExprSpanCoversOperands(t *testing.T) {
    left := &PawLiteral{Value: 1, NodeSpan: span(1, 1, 1, 2)}
    right := &PawLiteral{Value: 2, NodeSpan: span(1, 5, 1, 6)}
    expr := &BinaryExpr{Left: left, Op: token.PLUS, Right: right}

    got := expr.Span()
    if got.Start.Column != 1 || got.End.Column != 6 {
        t.Fatalf("span = %#v", got)
    }
}
```

- [ ] **Step 2: Run AST tests and verify failure**

```bash
go test ./internal/ast -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement nodes and span composition**

Use explicit structs rather than a generic map-shaped AST. Example:

```go
type BinaryExpr struct {
    Left  Expr
    Op    token.Kind
    Right Expr
}

func (*BinaryExpr) exprNode() {}
func (e *BinaryExpr) Span() source.Span {
    return source.Join(e.Left.Span(), e.Right.Span())
}
```

Type syntax:

```go
type NamedTypeExpr struct {
    Name token.Kind
    NodeSpan source.Span
}

type WrappedTypeExpr struct {
    Wrapper token.Kind // CLOWDER or MAYBE
    Inner TypeExpr
    NodeSpan source.Span
}
```

- [ ] **Step 4: Run AST tests**

```bash
go test ./internal/ast -v
```

Expected: PASS.

- [ ] **Step 5: Commit AST definitions**

```bash
git add internal/ast
git commit -m "feat: define Nyarn AST"
```

---

### Task 4: Implement the expression parser with v1 precedence and interpolation parsing

**Files:**
- Create: `internal/parser/parser.go`
- Create: `internal/parser/expression.go`
- Create: `internal/parser/parser_test.go`

**Interfaces:**
- Consumes: lexer tokens, AST nodes.
- Produces:
  - `parser.ParseExpressionForTest(filename, input string) (ast.Expr, []diagnostic.Diagnostic)`
  - internal recursive-descent methods matching spec precedence.
- `**` must be right-associative.
- Assignment is right-associative and target validation is deferred to semantic checking except for syntactically impossible targets if convenient.
- `..` has lower precedence than `+/-` and higher precedence than comparisons.

- [ ] **Step 1: Write failing precedence tests**

Required assertions. Define `parseExpr` in the same test file:

```go
func parseExpr(t *testing.T, input string) ast.Expr {
    t.Helper()
    expr, diags := ParseExpressionForTest("test.nyarn", input)
    if len(diags) != 0 {
        t.Fatalf("diagnostics = %#v", diags)
    }
    return expr
}

func TestPowerIsRightAssociative(t *testing.T) {
    expr := parseExpr(t, "2 ** 3 ** 2")
    outer := expr.(*ast.BinaryExpr)
    if outer.Op != token.STAR_STAR {
        t.Fatalf("outer op = %v", outer.Op)
    }
    if _, ok := outer.Right.(*ast.BinaryExpr); !ok {
        t.Fatal("right side must be nested power expression")
    }
}

func TestRangePrecedence(t *testing.T) {
    expr := parseExpr(t, "1 + 2 .. 10 - 1")
    r := expr.(*ast.RangeExpr)
    if _, ok := r.Start.(*ast.BinaryExpr); !ok {
        t.Fatal("range start must contain addition")
    }
    if _, ok := r.End.(*ast.BinaryExpr); !ok {
        t.Fatal("range end must contain subtraction")
    }
}
```

Also test:
- `nah a andpaw b orpaw c`
- call then index postfix chaining
- `[1, 2, 3]`
- `listen` with and without prompt
- `snatch cats` / `snatch cats, -1`
- `count cats`
- `sniffout cats, "Milo"`
- `paw("17")`
- grouping across newlines.

- [ ] **Step 2: Run parser expression tests and confirm failure**

```bash
go test ./internal/parser -run 'Test(Power|Range|Expression|List|Listen|Snatch|Count|Sniffout|Conversion)' -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement recursive-descent expression parsing**

Use methods in this exact precedence order:

```go
parseAssignment
parseOrpaw
parseAndpaw
parseEquality
parseComparison
parseRange
parseTerm
parseFactor
parsePower
parseUnary
parsePostfix
parsePrimary
```

`parsePower` must recurse into itself on the right side.

String interpolation:
- Lexer keeps raw decoded string text.
- Parser scans the string content for unescaped `{...}` regions after honoring doubled `{{` / `}}`.
- Each interpolation expression is lexed and parsed as a standalone expression with a span mapped back to the original string.
- Store the result as alternating text and expression segments in `ast.InterpolatedStringExpr`.

- [ ] **Step 4: Run expression parser tests**

```bash
go test ./internal/parser -run 'Test(Power|Range|Expression|List|Listen|Snatch|Count|Sniffout|Conversion|Interpolation)' -v
```

Expected: PASS.

- [ ] **Step 5: Commit expression parser**

```bash
git add internal/parser
git commit -m "feat: parse Nyarn expressions"
```

---

### Task 5: Implement statement parsing, blocks, functions, control flow, and type syntax

**Files:**
- Create: `internal/parser/statement.go`
- Modify: `internal/parser/parser.go`
- Modify: `internal/parser/parser_test.go`

**Interfaces:**
- Produces:
  - `func Parse(filename, input string) (*ast.Program, []diagnostic.Diagnostic)`
- Must parse all grammar in spec section 19.
- Must distinguish `hunt IDENTIFIER in expression {}` from `hunt expression {}` using lookahead without changing semantics.
- Must reject reserved future words with a specific “reserved/not yet supported” syntax diagnostic.

- [ ] **Step 1: Add failing program parser tests**

Canonical test source:

```nyarn
mew nyame: meow = "Mafu"
scratch lives: paw = 9

pounce greet(who: meow): maybe meow {
    sniff lives > 0 {
        purr "Henlo, {who}"
        gift who
    }
    nap {
        gift
    }
}

hunt cat in ["Milo", "Luna"] {
    sniff cat == "Milo" {
        prowl
    }
    purr cat
}
```

Assert exact statement node types and function parameter/type nodes.

Add a reserved keyword test:

```go
func TestReservedPspspsGetsSpecificDiagnostic(t *testing.T) {
    _, diags := Parse("test.nyarn", "pspsps \"math\"\n")
    if len(diags) == 0 || !strings.Contains(diags[0].Message, "reserved") {
        t.Fatalf("diagnostics = %#v", diags)
    }
}
```

- [ ] **Step 2: Run statement parser tests and verify failure**

```bash
go test ./internal/parser -run 'Test(ParseProgram|Reserved|Function|Hunt|Conditional|TypeSyntax)' -v
```

Expected: FAIL.

- [ ] **Step 3: Implement statement and type parsing**

Requirements:
- Consume optional blank `NEWLINE` tokens between statements.
- Within `(...)` and `[...]`, skip newline tokens.
- `gift` may omit its expression when followed by newline or `}`.
- `pounce` is parsed everywhere syntactically; Task 8 checker enforces top-level-only.
- `stash` is a statement.
- `snatch`, `count`, `sniffout`, `listen`, and conversions are expressions.
- Parse `clowder maybe meow` and `maybe clowder meow` recursively according to grammar.

- [ ] **Step 4: Run all parser tests**

```bash
go test ./internal/parser -v
```

Expected: PASS.

- [ ] **Step 5: Commit complete parser**

```bash
git add internal/parser
git commit -m "feat: parse Nyarn statements and types"
```

---

### Task 6: Implement the semantic type model and assignability rules

**Files:**
- Create: `internal/types/type.go`
- Create: `internal/types/type_test.go`

**Interfaces:**
- Produces:
  - `types.Type` interface or compact tagged struct.
  - Canonical values: `Paw`, `Whisker`, `Meow`, `Mood`, `EmptyBowl`, `Dynamic`.
  - Constructors `Clowder(element Type) Type`, `Maybe(inner Type) Type`.
  - `Assignable(from, to Type) bool`
  - `NumericJoin(a, b Type) (Type, bool)`
  - `EqualityCompatible(a, b Type) bool`
  - `Display(Type) string`
- `Dynamic` is checker-internal only and must never become a user-visible keyword.

- [ ] **Step 1: Write failing type-system unit tests**

```go
func TestNumericPromotion(t *testing.T) {
    got, ok := NumericJoin(Paw, Whisker)
    if !ok || !got.Equal(Whisker) {
        t.Fatalf("got %v, %v", got, ok)
    }
}

func TestEmptyBowlOnlyAssignableToMaybe(t *testing.T) {
    if Assignable(EmptyBowl, Meow) {
        t.Fatal("emptybowl must not assign to meow")
    }
    if !Assignable(EmptyBowl, Maybe(Meow)) {
        t.Fatal("emptybowl must assign to maybe meow")
    }
}

func TestClowderIsInvariantExceptNumericLiteralJoin(t *testing.T) {
    if Assignable(Clowder(Paw), Clowder(Whisker)) {
        t.Fatal("clowder assignment must not silently change element type")
    }
}
```

- [ ] **Step 2: Run type tests and confirm failure**

```bash
go test ./internal/types -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement canonical type values and compatibility**

Rules:
- `paw` assignable to `whisker`.
- Exact type assignability otherwise, plus `T` and `emptybowl` into `maybe T`.
- `maybe T` is not assignable to `T` without narrowing.
- `Dynamic` is compatible enough to defer checks until runtime.
- Clowders compare structurally by element type.
- `Display(Maybe(Clowder(Meow)))` returns `maybe clowder meow`.

- [ ] **Step 4: Run type tests**

```bash
go test ./internal/types -v
```

Expected: PASS.

- [ ] **Step 5: Commit semantic types**

```bash
git add internal/types
git commit -m "feat: add Nyarn semantic type model"
```

---

### Task 7: Implement checker scopes, declarations, identifier resolution, and mutability

**Files:**
- Create: `internal/checker/checker.go`
- Create: `internal/checker/scope.go`
- Create: `internal/checker/checker_test.go`

**Interfaces:**
- Produces:
  - `checker.New() *Checker`
  - `(*Checker).Check(program *ast.Program) []diagnostic.Diagnostic`
  - internal `Binding{Name string, Type types.Type, Mutable bool, DeclSpan source.Span, Global bool}`
  - lexical `Scope` with parent pointer and current-scope redeclaration detection.
- Must support child-scope shadowing and reject same-scope redeclaration.
- Must suggest nearby names for unknown identifiers.
- Must enforce mutable assignment targets.

- [ ] **Step 1: Write failing scope and mutability tests**

Required programs:
1. Child shadowing succeeds.
2. Same-scope redeclaration fails.
3. Unknown `nymae` suggests `nyame`.
4. Reassigning `mew` fails.
5. Assigning through `mew` list index fails.
6. Assigning through `scratch` list index passes.

Define shared checker test helpers in `checker_test.go`:

```go
func checkSource(t *testing.T, src string) []diagnostic.Diagnostic {
    t.Helper()
    program, parseDiags := parser.Parse("test.nyarn", src)
    if len(parseDiags) != 0 {
        t.Fatalf("parse diagnostics = %#v", parseDiags)
    }
    return New().Check(program)
}

func assertHasDiagnostic(t *testing.T, diags []diagnostic.Diagnostic, kind diagnostic.Kind, contains string) {
    t.Helper()
    for _, d := range diags {
        if d.Kind == kind && strings.Contains(d.Message, contains) {
            return
        }
    }
    t.Fatalf("missing %v diagnostic containing %q: %#v", kind, contains, diags)
}
```

Example:

```go
func TestMewIndexedMutationIsRejected(t *testing.T) {
    diags := checkSource(t, `
mew cats = ["Milo"]
cats[0] = "Luna"
`)
    assertHasDiagnostic(t, diags, diagnostic.Type, "cannot modify `cats`")
}
```

- [ ] **Step 2: Run checker tests and confirm failure**

```bash
go test ./internal/checker -run 'Test(Mew|Scratch|Shadow|Redeclare|Unknown)' -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement scopes and first-pass declaration checking**

Implementation rules:
- Build a global scope.
- Predeclare all top-level `pounce` names before checking executable top-level statements.
- Do not predeclare `mew`/`scratch`.
- For Task 7 only, declarations with an explicit annotation record that type immediately; unannotated declarations may use `Dynamic` as a temporary checker type until Task 8 adds full initializer inference.
- Each block creates a child scope.
- Function parameters bind in the function-local scope.
- A function scope has an explicit link to globals, not an enclosing function-local closure chain.
- Assignment resolves the root identifier and checks `Mutable`.
- An indexed assignment target is valid only when its root is an identifier bound by `scratch`.

- [ ] **Step 4: Run checker scope tests**

```bash
go test ./internal/checker -run 'Test(Mew|Scratch|Shadow|Redeclare|Unknown)' -v
```

Expected: PASS.

- [ ] **Step 5: Commit checker scope foundation**

```bash
git add internal/checker
git commit -m "feat: add Nyarn checker scopes"
```

---

### Task 8: Implement expression type checking, operators, clowders, `maybe`, and narrowing

**Files:**
- Create: `internal/checker/expression.go`
- Modify: `internal/checker/checker.go`
- Modify: `internal/checker/checker_test.go`

**Interfaces:**
- Produces internal `checkExpr(expr ast.Expr) types.Type`.
- Must infer literal/list types, numeric joins, built-in result types, index element types, range type, and dynamic fallback.
- Must track branch-local narrowing facts for `maybe T`.

- [ ] **Step 1: Add failing expression checker tests**

Required cases:
- `5 + 2.5` => `whisker`.
- `"lives: " + 9` => type error.
- `5 == "5"` => type error.
- `sniff 9 {}` => type error.
- `[1, 2.5]` => `clowder whisker`.
- `[1, "cat"]` => type error.
- bare `[]` => inference error.
- `mew x: maybe meow = emptybowl` => valid.
- `mew x: meow = emptybowl` => type error.
- narrowing inside `sniff x != emptybowl`.
- range bounds require `paw`.
- comparison chaining `0 < age < 18` must be rejected by semantic rules even if the parser builds nested comparisons.

- [ ] **Step 2: Run expression checker tests and confirm failure**

```bash
go test ./internal/checker -run 'Test(Type|Numeric|Clowder|Maybe|Narrow|Range|Comparison)' -v
```

Expected: FAIL.

- [ ] **Step 3: Implement expression checking**

Operator rules:
- `+`: numeric addition or `meow + meow`.
- `- * % **`: numeric only.
- `/`: numeric only, result `whisker`.
- `< <= > >=`: numeric or `meow`/`meow`; no `mood` or clowder ordering.
- `== !=`: same compatible type, numeric promotion, structural clowder equality; incompatible types error.
- `andpaw orpaw nah`: `mood` only.
- `IndexExpr`: target must be `clowder T` and index `paw`; `meow[index]` is not part of v1 indexing syntax unless explicitly added by a later spec revision.
- `RangeExpr`: both bounds `paw`, result checker-internal range type.

Narrowing:
- Recognize `identifier != emptybowl` and `identifier == emptybowl`.
- Apply facts only in the proven branch.
- On `nap`, invert the immediately corresponding `sniff` fact when valid.
- Do not persist narrowing after the conditional chain.

- [ ] **Step 4: Run expression checker tests**

```bash
go test ./internal/checker -run 'Test(Type|Numeric|Clowder|Maybe|Narrow|Range|Comparison)' -v
```

Expected: PASS.

- [ ] **Step 5: Commit expression checker**

```bash
git add internal/checker
git commit -m "feat: type-check Nyarn expressions"
```

---

### Task 9: Implement statement checking, functions, arity, return analysis, and loop legality

**Files:**
- Create: `internal/checker/statement.go`
- Modify: `internal/checker/checker.go`
- Modify: `internal/checker/checker_test.go`

**Interfaces:**
- Must enforce:
  - top-level-only `pounce`
  - exact function arity
  - typed parameters and optional dynamic parameters
  - typed return contracts
  - non-`maybe` typed functions cannot fall through
  - `gift` only in functions
  - `leap`/`prowl` only in `hunt`
  - condition expressions are `mood`
  - `stash` first operand is an identifier bound to a mutable `clowder`
  - `snatch` first operand is an identifier bound to a mutable `clowder`
  - `count`/`sniffout` operand rules

- [ ] **Step 1: Add failing function/control checker tests**

Required cases:

```nyarn
pounce add(a: paw, b: paw): paw {
    gift "cat"
}
```

must fail return type.

```nyarn
pounce get_life(x: paw): paw {
    sniff x > 0 {
        gift x
    }
}
```

must fail fallthrough.

```nyarn
pounce get_life(x: paw): maybe paw {
    sniff x > 0 {
        gift x
    }
}
```

must pass.

Also test:
- nested `pounce` fails;
- recursive `factorial` passes because functions are predeclared;
- wrong call arity fails;
- `leap` outside hunt fails;
- `prowl` outside hunt fails;
- function may mutate global `scratch`, may not mutate global `mew`.

- [ ] **Step 2: Run statement checker tests and confirm failure**

```bash
go test ./internal/checker -run 'Test(Function|Return|Arity|Loop|Global|Nested)' -v
```

Expected: FAIL.

- [ ] **Step 3: Implement statement checking and conservative return-flow analysis**

Use an internal flow result:

```go
type flow struct {
    AlwaysReturns bool
}
```

Rules:
- A `gift` statement always returns.
- A block always returns only if execution cannot reach its end.
- An `sniff` chain always returns only when it has a `nap` branch and every branch always returns.
- `hunt` never counts as guaranteed return in v1, because the checker does not prove loop termination.
- Dynamic return functions skip a single static return contract but still validate statements inside them.

- [ ] **Step 4: Run complete checker tests**

```bash
go test ./internal/checker -v
```

Expected: PASS.

- [ ] **Step 5: Commit statement checker**

```bash
git add internal/checker
git commit -m "feat: validate Nyarn functions and control flow"
```

---

### Task 10: Implement runtime values, environments, ranges, functions, and control signals

**Files:**
- Create: `internal/runtime/value.go`
- Create: `internal/runtime/environment.go`
- Create: `internal/runtime/function.go`
- Create: `internal/runtime/range.go`
- Create: `internal/runtime/signal.go`
- Create: `internal/runtime/runtime_test.go`

**Interfaces:**
- Produces:
  - `runtime.Value` tagged representation
  - constructors/accessors for paw, whisker, meow, mood, clowder, emptybowl, function, range
  - `Environment.Define`, `Get`, `Assign`
  - `Function{Name, Params, Body, ReturnType, DeclSpan}`
  - lazy `Range{Start int64, End int64}`
  - control signals `ReturnSignal`, `BreakSignal`, `ContinueSignal`, `HissSignal`
- Use checked integer operations or explicit overflow detection for `paw`.

- [ ] **Step 1: Write failing runtime unit tests**

Required:
- nested environment lookup and shadowing;
- assignment updates nearest mutable binding;
- immutable assignment rejected;
- lazy range iterates `0,1,2` for `0..3`;
- clowder display uses canonical Nyarn textual values;
- mood display is `purring`/`hissing`;
- emptybowl display is `emptybowl`.

- [ ] **Step 2: Run runtime tests and confirm failure**

```bash
go test ./internal/runtime -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement runtime primitives**

Use a stable enum/tag rather than reflection-based type switching on arbitrary Go values.

`clowder` values must use value semantics. Do not expose a mutable Go slice that can be aliased across `mew` and `scratch` bindings. Indexed assignment, `stash`, and `snatch` create an updated clowder value and write it back through the mutable root binding; nested index mutation clones the clowder path from the root. This preserves `mew` immutability even when one binding was initialized from another clowder.

Canonical display examples:

```text
Paw(9) -> 9
Whisker(2.5) -> 2.5
Meow("cat") -> cat
Mood(true) -> purring
Mood(false) -> hissing
EmptyBowl -> emptybowl
Clowder([Paw(1), Paw(2)]) -> [1, 2]
```

- [ ] **Step 4: Run runtime tests**

```bash
go test ./internal/runtime -v
```

Expected: PASS.

- [ ] **Step 5: Commit runtime foundation**

```bash
git add internal/runtime
git commit -m "feat: add Nyarn runtime values and scopes"
```

---

### Task 11: Implement interpreter expressions, arithmetic, indexing, interpolation, and ranges

**Files:**
- Create: `internal/interpreter/interpreter.go`
- Create: `internal/interpreter/expression.go`
- Create: `internal/interpreter/interpreter_test.go`

**Interfaces:**
- Produces:
  - `interpreter.New(stdin io.Reader, stdout io.Writer) *Interpreter`
  - `Interpreter.EvalExpr(ast.Expr, *runtime.Environment) (runtime.Value, *diagnostic.Diagnostic)`
- Assumes AST already passed the checker, but dynamic parameters can still trigger runtime type failures.
- No raw panic may escape user expressions.

- [ ] **Step 1: Write failing expression interpreter tests**

Execute parsed+checked snippets through a test helper and assert:
- `purr 5 / 2` => `2.5\n`
- `purr 2 ** 3 ** 2` => `512\n`
- `purr [1, 2, 3][-1]` => `3\n`
- out-of-range index => runtime HISS
- `purr "{1 + 2}"` => `3\n`
- `purr 5 == 5.0` => `purring\n`
- division by zero => runtime diagnostic, no panic.

- [ ] **Step 2: Run interpreter expression tests and verify failure**

```bash
go test ./internal/interpreter -run 'Test(Arithmetic|Power|Index|Interpolation|Equality|Division)' -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement expression evaluation**

Requirements:
- Preserve `paw` when integer operations remain representable; otherwise use `whisker` where spec says so or HISS on integer overflow where no float promotion is semantically justified.
- `/` returns `whisker`.
- `%` supports numeric operands and HISSes on zero divisor.
- `**` follows spec result rule.
- `clowder` equality is structural and order-sensitive.
- `meow` ordering is lexical.
- negative clowder indices normalize from the end.
- interpolation calls runtime display formatting for expression segments.
- range values remain lazy.

- [ ] **Step 4: Run expression interpreter tests**

```bash
go test ./internal/interpreter -run 'Test(Arithmetic|Power|Index|Interpolation|Equality|Division|Range)' -v
```

Expected: PASS.

- [ ] **Step 5: Commit expression interpreter**

```bash
git add internal/interpreter
git commit -m "feat: evaluate Nyarn expressions"
```

---

### Task 12: Implement statements, control flow, functions, hoisting, recursion, and Nyarn stack traces

**Files:**
- Create: `internal/interpreter/statement.go`
- Modify: `internal/interpreter/interpreter.go`
- Modify: `internal/interpreter/interpreter_test.go`

**Interfaces:**
- Produces:
  - `Interpreter.Execute(program *ast.Program) []diagnostic.Diagnostic`
- Execution has two passes:
  1. register all top-level `pounce` declarations;
  2. execute non-function top-level statements in source order.
- Function calls create call frames used for Nyarn stack traces.
- `gift`, `leap`, `prowl`, and `hiss` use dedicated internal signals.

- [ ] **Step 1: Write failing control-flow and function tests**

Required source programs:
- hoisted function called before declaration;
- recursive factorial returns 120;
- `sniff/peek/nap` chooses exactly one branch;
- `hunt lives > 0` terminates;
- `hunt i in 0..3` prints `0 1 2`;
- string iteration handles Unicode runes;
- `leap` exits nearest hunt;
- `prowl` skips to next iteration;
- explicit `hiss` produces non-catchable runtime diagnostic;
- nested function calls produce stack frames in top-to-bottom source order.

- [ ] **Step 2: Run statement interpreter tests and confirm failure**

```bash
go test ./internal/interpreter -run 'Test(Hoist|Recursion|Sniff|Hunt|Leap|Prowl|Hiss|Stack)' -v
```

Expected: FAIL.

- [ ] **Step 3: Implement statement execution and call frames**

Function call algorithm must match the spec:
1. resolve function;
2. validate runtime arity defensively;
3. push call frame;
4. create function-local environment with access to globals;
5. bind parameters;
6. execute body;
7. convert `ReturnSignal` to value;
8. default to `emptybowl`;
9. pop frame on all paths.

Use `defer` only for interpreter-internal cleanup; convert unexpected panics at the runner boundary, not by swallowing ordinary control signals.

- [ ] **Step 4: Run all interpreter control-flow tests**

```bash
go test ./internal/interpreter -run 'Test(Hoist|Recursion|Sniff|Hunt|Leap|Prowl|Hiss|Stack)' -v
```

Expected: PASS.

- [ ] **Step 5: Commit statements and functions**

```bash
git add internal/interpreter
git commit -m "feat: execute Nyarn control flow and functions"
```

---

### Task 13: Implement core built-ins: I/O, conversions, and clowder operations

**Files:**
- Create: `internal/interpreter/builtin.go`
- Modify: `internal/interpreter/expression.go`
- Modify: `internal/interpreter/statement.go`
- Modify: `internal/interpreter/interpreter_test.go`

**Interfaces:**
- Must implement exactly:
  - `purr expr`
  - `listen` and `listen prompt`
  - `paw(expr)`, `whisker(expr)`, `meow(expr)`, `mood(expr)`
  - `stash clowder, value`
  - `snatch clowder` and `snatch clowder, index`
  - `count value`
  - `sniffout haystack, needle`

- [ ] **Step 1: Write failing built-in tests**

Required behavior:
- `listen "Nyame: "` writes prompt without newline and strips trailing input newline.
- `paw("17") == 17`.
- `paw(2.0) == 2`.
- `paw(2.5)` HISSes.
- `mood("purring")` => purring; other strings HISS.
- `meow(purring)` => `"purring"` as a meow value.
- `stash` mutates a `scratch clowder`.
- `snatch` returns removed value and supports negative index.
- `count "gatão"` counts Unicode characters.
- `sniffout "Nyarn", "yar"` => purring.
- mutating a `mew` clowder is rejected before runtime by checker.

- [ ] **Step 2: Run built-in tests and confirm failure**

```bash
go test ./internal/interpreter -run 'Test(Listen|Convert|Stash|Snatch|Count|Sniffout|Purr)' -v
```

Expected: FAIL.

- [ ] **Step 3: Implement built-ins with checker-compatible runtime validation**

Canonical conversion rules must match spec section 13.3 exactly.

For `stash` and `snatch`, require an identifier target, resolve that mutable binding, build a new clowder value, and assign the replacement back to the binding. Do not mutate a shared backing slice.

`count meow` must use rune count, not byte length.

- [ ] **Step 4: Run built-in and full interpreter tests**

```bash
go test ./internal/interpreter -v
```

Expected: PASS.

- [ ] **Step 5: Commit built-ins**

```bash
git add internal/interpreter
git commit -m "feat: add Nyarn core built-ins"
```

---

### Task 14: Build the runner pipeline, panic boundary, exit classification, and file execution

**Files:**
- Create: `internal/runner/runner.go`
- Create: `internal/runner/runner_test.go`

**Interfaces:**
- Produces:

```go
type Result struct {
    ExitCode int
    Diagnostics []diagnostic.Diagnostic
}

func RunSource(filename, sourceText string, stdin io.Reader, stdout, stderr io.Writer) Result
func RunFile(path string, stdin io.Reader, stdout, stderr io.Writer) Result
```

- Pipeline is lex → parse → check → interpret.
- If any stage emits diagnostics, later stages do not run.
- A recovered unexpected Go panic becomes an internal diagnostic and nonzero exit.

- [ ] **Step 1: Write failing runner tests**

Required:
- syntax error => exit 2 and no output from executable statements;
- checker/type error => exit 3 and no execution;
- explicit `hiss`/division zero => exit 1;
- missing file => exit 4;
- a file path without the `.nyarn` extension => exit 4;
- successful hello => exit 0;
- injected panic from a test-only interpreter hook => internal diagnostic, no raw Go stack unless debug mode is enabled.

- [ ] **Step 2: Run runner tests and confirm failure**

```bash
go test ./internal/runner -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement runner pipeline and diagnostic rendering**

Render all diagnostics to `stderr`. User program `purr` output goes to `stdout`.

`RunFile` must reject a non-`.nyarn` path with an I/O/CLI diagnostic and exit code 4 before reading or executing it.

Map errors:
- lexer/parser syntax => 2
- checker/name/type => 3
- runtime/explicit hiss => 1
- file/I/O => 4
- internal => 1 unless a later design revision assigns a dedicated code.

- [ ] **Step 4: Run runner tests**

```bash
go test ./internal/runner -v
```

Expected: PASS.

- [ ] **Step 5: Commit runner**

```bash
git add internal/runner
git commit -m "feat: add Nyarn execution pipeline"
```

---

### Task 15: Implement the persistent multiline REPL

**Files:**
- Create: `internal/repl/repl.go`
- Create: `internal/repl/repl_test.go`
- Modify: `internal/checker/checker.go`
- Modify: `internal/checker/scope.go`
- Modify: `internal/runtime/environment.go`

**Interfaces:**
- Produces:
  - `repl.Run(stdin io.Reader, stdout, stderr io.Writer) int`
  - `(*checker.Checker).Clone() *checker.Checker`
  - `(*runtime.Environment).Clone() *runtime.Environment`
- Global semantic scope, runtime environment, and top-level functions persist across successful entries.
- User errors do not terminate the REPL.
- Incomplete `{`, `[`, `(` input continues with `... ` prompt.
- A submitted unit is lexed, parsed, checked, and executed before accepting the next unit.

- [ ] **Step 1: Write failing REPL transcript tests**

Input:

```text
mew x = 9
purr x
scratch y = 2
y = 7
purr y
```

must include `9` and `7`.

Multiline input:

```text
pounce greet(nyame: meow) {
purr "Henlo, {nyame}"
}
greet("Mafu")
```

must define the function and print `Henlo, Mafu`.

Error recovery transcript:
- submit `purr 1 / 0`;
- then submit `purr "still here"`;
- output must contain the HISS diagnostic and `still here`.

Transactional-state transcript:
- submit a multiline unit that declares `mew doomed = 1` and then HISSes;
- next submit `purr doomed`;
- the second submission must produce an unknown-name HISS because failed units do not commit semantic/runtime state.

- [ ] **Step 2: Run REPL tests and confirm failure**

```bash
go test ./internal/repl -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement cloneable checker/runtime state and persistent REPL sessions**

Create a session object that owns:
- persistent checker state/global semantic scope;
- persistent runtime global environment and function declarations;
- accumulated source-name counter such as `<repl:1>`, `<repl:2>`;
- input balance state.

`Checker.Clone` must deep-copy scopes/binding metadata used by the persistent global checker state. `Environment.Clone` must copy the binding map and clowder values using Nyarn value semantics so a trial submission cannot mutate committed state through a shared Go slice.

Treat each submitted REPL unit transactionally: clone the checker global state and runtime global environment, check and execute against the clones, and commit both clones only when the unit succeeds. A syntax/checker/runtime HISS must leave the previously committed REPL state unchanged. This keeps semantic and runtime state synchronized even when an error occurs before or after a declaration.

Do not concatenate all previous REPL source and re-execute it. Execute only the new submission against the persistent committed session state.

- [ ] **Step 4: Run REPL plus checker/runtime regression tests**

```bash
go test ./internal/repl ./internal/checker ./internal/runtime -v
```

Expected: PASS.

- [ ] **Step 5: Commit REPL**

```bash
git add internal/repl internal/checker internal/runtime
git commit -m "feat: add transactional Nyarn REPL"
```

---

### Task 16: Implement the CLI entry point and exact command behavior

**Files:**
- Create: `cmd/nyarn/main.go`
- Create: `cmd/nyarn/main_test.go`

**Interfaces:**
- `nyarn file.nyarn` executes a file.
- `nyarn run file.nyarn` executes the same file.
- `nyarn` starts the REPL.
- invalid argument shapes return exit 4 and a concise usage diagnostic.

- [ ] **Step 1: Write failing CLI argument tests**

Extract argument dispatch into a testable function:

```go
func run(args []string, stdin io.Reader, stdout, stderr io.Writer) int
```

Required cases:
- `[]` calls `repl.Run`;
- `["hello.nyarn"]` delegates to file runner;
- `["run", "hello.nyarn"]` delegates to file runner;
- `["run"]` => 4;
- `["wat", "a", "b"]` => 4.

- [ ] **Step 2: Run CLI tests and confirm failure**

```bash
go test ./cmd/nyarn -v
```

Expected: compilation failure.

- [ ] **Step 3: Implement CLI dispatch**

`main()` must do only:

```go
func main() {
    os.Exit(run(os.Args[1:], os.Stdin, os.Stdout, os.Stderr))
}
```

Do not bury language logic in `cmd/nyarn`.

- [ ] **Step 4: Run CLI tests and build binary**

```bash
go test ./cmd/nyarn -v
go build ./cmd/nyarn
```

Expected: PASS and successful build.

- [ ] **Step 5: Commit CLI**

```bash
git add cmd/nyarn
git commit -m "feat: add Nyarn CLI"
```

---

### Task 17: Add canonical examples and end-to-end conformance tests

**Files:**
- Create: `examples/hello.nyarn`
- Create: `examples/loops.nyarn`
- Create: `examples/functions.nyarn`
- Create: `examples/clowder.nyarn`
- Create: `tests/conformance_test.go`

**Interfaces:**
- Tests the public runner rather than internal parser/checker APIs.
- Canonical examples must remain valid against `docs/SPEC.md`.

- [ ] **Step 1: Add failing conformance tests before example files exist**

Use a table:

```go
func TestCanonicalExamples(t *testing.T) {
    cases := []struct{
        file string
        wantContains []string
    }{
        {"../examples/hello.nyarn", []string{"Henlo from Nyarn!"}},
        {"../examples/functions.nyarn", []string{"120"}},
        {"../examples/loops.nyarn", []string{"0", "1", "2"}},
        {"../examples/clowder.nyarn", []string{"Amora"}},
    }

    for _, tc := range cases {
        t.Run(tc.file, func(t *testing.T) {
            var out, errOut bytes.Buffer
            result := runner.RunFile(tc.file, strings.NewReader(""), &out, &errOut)
            if result.ExitCode != 0 {
                t.Fatalf("exit=%d stderr=%s", result.ExitCode, errOut.String())
            }
            for _, want := range tc.wantContains {
                if !strings.Contains(out.String(), want) {
                    t.Fatalf("missing %q in %q", want, out.String())
                }
            }
        })
    }
}
```

Also add negative conformance programs in-memory for:
- truthiness rejection;
- heterogeneous clowder rejection;
- wrong arity;
- `mew` mutation;
- out-of-bounds runtime error;
- reserved `pspsps`.

- [ ] **Step 2: Run conformance tests and confirm failure**

```bash
go test ./tests -v
```

Expected: FAIL because example files are missing.

- [ ] **Step 3: Add canonical examples**

`examples/hello.nyarn`:

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

`examples/functions.nyarn`:

```nyarn
pounce factorial(n: paw): paw {
    sniff n <= 1 {
        gift 1
    }

    gift n * factorial(n - 1)
}

purr factorial(5)
```

`examples/loops.nyarn`:

```nyarn
hunt i in 0..5 {
    sniff i == 3 {
        leap
    }
    purr i
}
```

`examples/clowder.nyarn`:

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]
stash cats, "Amora"
purr cats[-1]
purr count cats
```

- [ ] **Step 4: Run all tests and race detector**

```bash
go test ./... -v
go test -race ./...
```

Expected: PASS.

- [ ] **Step 5: Commit examples and conformance suite**

```bash
git add examples tests
git commit -m "test: add Nyarn v1 conformance examples"
```

---

### Task 18: Add the MIT license and replace the placeholder README with the unserious public README

**Files:**
- Create: `LICENSE`
- Modify: `README.md`
- Test: `tests/conformance_test.go` for README code sample synchronization if practical.

**Interfaces:**
- README is public-facing and unserious in voice, but every technical claim must match `docs/SPEC.md`.
- README must link to `docs/SPEC.md`.
- README must show installation/build instructions that work from a clean clone.
- Do not advertise unimplemented v2 features as available.

- [ ] **Step 1: Write the README content using only implemented features**

Required opening:

```md
# Nyarn

> **Nyarn is what happens when someone gives a cat access to language design.**

Is this necessary? No.

Is that going to stop us? Also no.
```

Required sections:
- a canonical Nyarn sample;
- “Is Nyarn a joke?” / “Is Nyarn an actual programming language?”;
- implemented features;
- build/run instructions;
- keyword taste-test;
- diagnostics example;
- link to language spec;
- v1 non-goals / roadmap;
- MIT license note.

Include:

```md
## Is Nyarn a joke?

Yes.

## Is Nyarn an actual programming language?

Also yes.

This has created several problems.
```

And a diagnostics sample in the approved style:

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

- [ ] **Step 2: Add the MIT license**

Use the standard MIT license text with the repository owner's chosen copyright line. If no explicit display name is documented in the repository, use:

```text
Copyright (c) 2026 mafuzyk
```

- [ ] **Step 3: Verify README commands against the real binary**

Run:

```bash
go build -o ./nyarn ./cmd/nyarn
./nyarn examples/hello.nyarn
printf 'purr "mrrp"\n' | ./nyarn
```

Expected:
- hello example prints `Henlo from Nyarn!`;
- piped REPL input includes `mrrp`;
- no README command references missing tooling.

- [ ] **Step 4: Run the full suite one final time**

```bash
go test ./...
go test -race ./...
go vet ./...
```

Expected: all commands succeed.

- [ ] **Step 5: Commit public project surface**

```bash
git add README.md LICENSE
git commit -m "docs: introduce Nyarn to the world"
```

---

## Final Verification Gate

Before calling Nyarn v1 implementation-complete, run exactly:

```bash
gofmt -w $(find cmd internal tests -type f -name '*.go')
go test ./...
go test -race ./...
go vet ./...
go build -o ./nyarn ./cmd/nyarn
./nyarn examples/hello.nyarn
./nyarn run examples/functions.nyarn
```

Expected outputs include:

```text
Henlo from Nyarn!
120
```

Then manually verify these failure modes:

```bash
cat > /tmp/type-error.nyarn <<'NYARN'
mew lives = 9
sniff lives {
    purr "nope"
}
NYARN
./nyarn /tmp/type-error.nyarn
echo $?
```

Expected exit code: `3`.

```bash
cat > /tmp/runtime-error.nyarn <<'NYARN'
purr 1 / 0
NYARN
./nyarn /tmp/runtime-error.nyarn
echo $?
```

Expected exit code: `1`.

```bash
cat > /tmp/reserved.nyarn <<'NYARN'
pspsps "math"
NYARN
./nyarn /tmp/reserved.nyarn
echo $?
```

Expected exit code: `2`.

The final implementation is acceptable only when:
- all automated tests pass;
- race detector passes;
- `go vet` passes;
- both CLI file forms work;
- REPL state persists across submissions;
- diagnostics show useful technical context before cat humor;
- no raw Go panic leaks for user-caused Nyarn errors;
- every v1 feature documented in `docs/SPEC.md` is either covered by unit/conformance tests or explicitly exercised by canonical examples.
