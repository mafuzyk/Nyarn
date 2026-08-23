---
title: Nyarn v1 — Guia Completo
aliases:
  - Nyarn v1 Reference
  - Referência Nyarn v1
  - Manual Nyarn v1
  - Nyarn Complete Guide
tags:
  - nyarn
  - programming-language
  - reference
  - documentation
status: design-aprovado
version: v1
language: pt-BR
cssclasses:
  - nyarn-doc
  - nyarn-reference
---

# 🧶 Nyarn v1 — Guia Completo

> [!abstract] O que é esta nota?
> Esta é a **referência completa e legível da Nyarn v1** para uso no Obsidian.
>
> Para aprender do zero, comece por [[Nyarn v1 - Introdução]]. Para comportamento normativo e detalhes de implementação que precisem ser decididos sem ambiguidade, a fonte de verdade continua sendo [[SPEC|Nyarn v1 — Especificação]].

> [!quote]
> **Nyarn is what happens when someone gives a cat access to language design.**

---

# 🗺️ Mapa da nota

- [[#1. Visão geral]]
- [[#2. Arquivos e sintaxe básica]]
- [[#3. Identificadores e palavras reservadas]]
- [[#4. Sistema de tipos]]
- [[#5. Declarações, mutabilidade e escopo]]
- [[#6. Booleanos e lógica]]
- [[#7. Controle de fluxo]]
- [[#8. Ranges]]
- [[#9. Funções]]
- [[#10. Operadores e números]]
- [[#11. Strings]]
- [[#12. Clowders]]
- [[#13. Built-ins]]
- [[#14. maybe e emptybowl]]
- [[#15. Conversões]]
- [[#16. Erros e HISS]]
- [[#17. CLI]]
- [[#18. REPL]]
- [[#19. Precedência de operadores]]
- [[#20. Runtime e modelo de execução]]
- [[#21. Coisas que NÃO existem na v1]]
- [[#22. Exemplos completos]]
- [[#23. Referência rápida]]

---

# 1. Visão geral

Nyarn v1 é uma linguagem:

- interpretada;
- implementada em Go;
- executada por tree-walk interpreter;
- single-file;
- com tipagem opcional;
- com inferência de tipos;
- com checker semântico separado do interpretador;
- com escopo léxico;
- imutável por padrão;
- com funções top-level e recursão;
- com listas homogêneas;
- com diagnósticos estruturados.

Pipeline conceitual:

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

> [!important] Regra central
> Se um erro puder ser conhecido estaticamente, a Nyarn deve rejeitá-lo **antes da execução normal**.

---

# 2. Arquivos e sintaxe básica

Extensão:

```text
.nyarn
```

Statements normalmente terminam com quebra de linha.

```nyarn
mew nyame = "Nyarn"
purr nyame
```

Não existem `;` na v1.

Linhas vazias são ignoradas.

Dentro de agrupamentos, quebras de linha podem aparecer livremente:

```nyarn
mew result = (
    10 +
    20 +
    30
)
```

```nyarn
mew cats: clowder meow = [
    "Milo",
    "Luna",
    "Dexter"
]
```

## Comentários

Comentários usam `#`.

```nyarn
# comentário inteiro
mew nyame = "Mafu"  # comentário inline
```

Não existe comentário multilinha na v1.

## Blocos

Blocos usam `{}`:

```nyarn
sniff hungry {
    purr "FOOD"
}
```

Cada bloco cria um escopo léxico filho.

---

# 3. Identificadores e palavras reservadas

Identificadores são case-sensitive.

```text
nyame
Nyame
NYAME
```

são nomes diferentes.

Podem conter:

- letras Unicode;
- `_`;
- dígitos após o primeiro caractere.

Exemplos válidos:

```nyarn
mew gatinho = "mrrp"
mew coração = purring
mew cat_2 = "Luna"
```

Emoji não são válidos como caracteres de identificador.

## Keywords da v1

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

Keywords não podem ser usadas como identificadores.

> [!fun]
> `nyame` **não** é keyword.
>
> É cultura.

## Reservadas para versões futuras

```text
pspsps
breed
carefully
hissback
```

Planejamento atual:

| Palavra | Futuro significado |
|---|---|
| `pspsps` | imports / módulos |
| `breed` | records / structs |
| `carefully` | bloco de captura de erro |
| `hissback` | handler do erro |

Essas palavras já são reservadas na v1 mesmo sem implementação.

---

# 4. Sistema de tipos

## Tipos públicos

| Tipo | Significado |
|---|---|
| `paw` | inteiro |
| `whisker` | float |
| `meow` | string |
| `mood` | boolean |
| `clowder T` | lista homogênea de `T` |
| `maybe T` | `T` ou `emptybowl` |

Valores especiais:

```nyarn
purring    # true
hissing    # false
emptybowl  # ausência / null
```

Não existe `any` público na v1.

O checker pode usar um tipo dinâmico interno para parâmetros sem anotação, mas o usuário não escreve `any`.

## Inferência

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

## Anotação explícita

```nyarn
mew age: paw = 17
mew height: whisker = 1.72
mew nyame: meow = "Mafu"
mew sleepy: mood = purring
```

A anotação é um contrato.

```nyarn
mew age: paw = "seventeen"
```

→ HISS de tipo.

---

# 5. Declarações, mutabilidade e escopo

## `mew`

Declara binding imutável:

```nyarn
mew nyame = "Mafu"
```

Não pode reatribuir:

```nyarn
nyame = "Nyarn"  # HISS
```

Também não pode mutar uma clowder por esse root binding:

```nyarn
mew cats = ["Milo", "Luna"]
cats[0] = "Dexter"  # HISS
```

## `scratch`

Declara binding mutável:

```nyarn
scratch lives = 9
lives = 8
```

```nyarn
scratch cats = ["Milo", "Luna"]
cats[0] = "Dexter"
```

Na v1, mutabilidade é propriedade do **binding**, não do valor runtime em si.

## Shadowing

Permitido entre escopos:

```nyarn
mew nyame = "Mafu"

sniff purring {
    mew nyame = "Nyarn"
    purr nyame  # Nyarn
}

purr nyame      # Mafu
```

Redeclaração no mesmo escopo é proibida:

```nyarn
mew nyame = "Mafu"
mew nyame = "Nyarn"  # HISS
```

---

# 6. Booleanos e lógica

Tipo:

```text
mood
```

Valores:

```text
purring -> true
hissing -> false
```

Não existe truthiness geral.

Só `mood` pode ser usado diretamente como condição.

```nyarn
mew hungry = purring

sniff hungry {
    purr "FOOD"
}
```

Isto é inválido:

```nyarn
sniff 9 {
    purr "???"
}
```

## Operadores lógicos

```text
andpaw  AND
orpaw   OR
nah     NOT
```

```nyarn
sniff hungry andpaw nah sleepy {
    purr "kitchen destruction authorized"
}
```

Todos exigem operandos `mood` e retornam `mood`.

---

# 7. Controle de fluxo

## Condicionais

```text
sniff -> if
peek  -> else if
nap   -> else
```

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

Regras:

- um `sniff` inicial;
- zero ou mais `peek`;
- zero ou um `nap` final;
- `sniff` e `peek` exigem `mood`.

## `hunt` condicional

```nyarn
hunt lives > 0 {
    lives = lives - 1
}
```

A condição é reavaliada a cada iteração e precisa ser `mood`.

## `hunt ... in ...`

```nyarn
hunt cat in cats {
    purr cat
}
```

Iteráveis válidos:

- `clowder T`;
- `meow`;
- range.

A variável de iteração é imutável.

Strings são percorridas por caracteres Unicode, não bytes crus.

## `leap`

Equivale a `break`:

```nyarn
hunt cat in cats {
    sniff cat == "Dexter" {
        leap
    }
}
```

Fora de `hunt` → erro do checker.

## `prowl`

Equivale a `continue`:

```nyarn
hunt cat in cats {
    sniff cat == "Luna" {
        prowl
    }

    purr cat
}
```

Fora de `hunt` → erro do checker.

---

# 8. Ranges

Sintaxe:

```nyarn
0..10
```

É half-open:

```text
0..10 = 0,1,2,3,4,5,6,7,8,9
```

```nyarn
hunt i in 0..3 {
    purr i
}
```

Saída:

```text
0
1
2
```

Regras da v1:

- bounds são `paw`;
- ranges são lazy em runtime;
- não existe step customizado.

---

# 9. Funções

Declaração:

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}
```

## Parâmetros

Tipos são opcionais:

```nyarn
pounce greet(nyame) {
    purr "Henlo, {nyame}"
}
```

Argument count é exato.

Não existem default arguments na v1.

## Retorno

```nyarn
gift expression
```

ou:

```nyarn
gift
```

`gift` sem valor retorna `emptybowl`.

Chegar ao fim da função também retorna `emptybowl`.

## Return type

Opcional:

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}
```

Se existe anotação de retorno, todos os caminhos relevantes devem obedecer.

Inválido:

```nyarn
pounce answer(): paw {
    gift "cat"
}
```

Também inválido:

```nyarn
pounce get_life(x: paw): paw {
    sniff x > 0 {
        gift x
    }
}
```

porque existe caminho que cai no fim e retorna `emptybowl`.

Válido:

```nyarn
pounce get_life(x: paw): maybe paw {
    sniff x > 0 {
        gift x
    }
}
```

## Top-level only

Na v1, `pounce` só aparece no top-level.

Sem nested functions.

Sem closures.

## Hoisting

Funções são registradas antes dos statements top-level executarem:

```nyarn
greet("Mafu")

pounce greet(nyame: meow) {
    purr "Henlo, {nyame}"
}
```

funciona.

Bindings normais não são hoisted.

## Recursão

Suportada:

```nyarn
pounce factorial(n: paw): paw {
    sniff n <= 1 {
        gift 1
    }

    gift n * factorial(n - 1)
}
```

## Globals

Funções podem ler globals.

Podem modificar globals apenas quando declaradas com `scratch`.

```nyarn
scratch counter = 0

pounce bump() {
    counter = counter + 1
}
```

---

# 10. Operadores e números

## Aritméticos

```text
+  -  *  /  %  **
```

## Comparação

```text
==  !=  >  >=  <  <=
```

## Promoção numérica

Existe uma única promoção implícita:

```text
paw -> whisker
```

Tabela básica:

| Operação | Resultado |
|---|---|
| `paw + paw` | `paw` |
| `paw + whisker` | `whisker` |
| `whisker + paw` | `whisker` |
| `whisker + whisker` | `whisker` |

A promoção vale também para comparações numéricas.

## Divisão

`/` sempre retorna `whisker`.

```nyarn
purr 5 / 2
# 2.5
```

Divisão por zero → HISS runtime.

Não existe divisão inteira dedicada na v1.

## Módulo

`%` aceita números.

- `paw % paw` → `paw`;
- presença de `whisker` → promoção e `whisker`.

Módulo por zero → HISS runtime.

## Potência

`**` é right-associative:

```nyarn
2 ** 3 ** 2
```

é:

```text
2 ** (3 ** 2)
```

Se ambos forem `paw` e o resultado for exatamente representável como inteiro pelas regras checked da implementação, pode permanecer `paw`; caso contrário vira `whisker`.

## Igualdade entre tipos incompatíveis

```nyarn
5 == "5"
```

não retorna `hissing`.

É erro de tipo.

Mas:

```nyarn
5 == 5.0
```

→ `purring`.

## Ordering

Números suportam ordering.

Strings suportam ordering lexical:

```nyarn
"Amora" < "Dexter"
```

`mood` aceita apenas `==` e `!=`.

`clowder` aceita apenas `==` e `!=` por comparação estrutural.

## Comparações encadeadas

Não existem na v1:

```nyarn
0 < age < 18
```

é inválido.

Use:

```nyarn
age > 0 andpaw age < 18
```

---

# 11. Strings

Somente aspas duplas:

```nyarn
mew nyame = "Mafu"
```

Escapes:

```text
\n   newline
\t   tab
\"   aspas
\\   barra invertida
```

## Interpolação

```nyarn
mew nyame = "Mafu"
mew lives = 9

purr "{nyame} has {lives} lives"
purr "next life: {lives - 1}"
```

Aceita expressões completas.

Chaves literais:

```nyarn
purr "{{hello}}"
```

→ `{hello}`

## Concatenação

```nyarn
"nya" + "rn"
```

→ `"nyarn"`

Mas:

```nyarn
"lives: " + 9
```

→ HISS.

Use interpolação ou `meow(...)`.

---

# 12. Clowders

Tipo de lista:

```nyarn
clowder T
```

Exemplo:

```nyarn
mew cats: clowder meow = ["Milo", "Luna"]
```

São homogêneas.

```nyarn
mew values = [1, "two", purring]
```

→ HISS.

Numeric promotion pode unificar:

```nyarn
mew numbers = [1, 2.5]
```

→ `clowder whisker`.

## Lista vazia

```nyarn
mew cats = []
```

não pode inferir tipo.

Use:

```nyarn
mew cats: clowder meow = []
```

## Indexação

```nyarn
cats[0]
cats[-1]
```

Índices negativos funcionam estilo Python.

Out-of-bounds → HISS runtime.

## Mutação

```nyarn
scratch cats = ["Milo"]
cats[0] = "Luna"
```

Válido.

Com `mew` → HISS.

## Igualdade estrutural

```nyarn
[1, 2] == [1, 2]
```

→ `purring`.

Ordem importa:

```nyarn
[1, 2] == [2, 1]
```

→ `hissing`.

Slicing não existe na v1.

---

# 13. Built-ins

## `purr`

Imprime exatamente uma expressão e adiciona newline.

```nyarn
purr "hello"
purr 42
purr purring
purr emptybowl
```

Display não equivale a conversão de tipo.

## `listen`

Retorna `meow`.

Com prompt:

```nyarn
mew nyame = listen "Nyame: "
```

Sem prompt:

```nyarn
mew input = listen
```

Prompt precisa ser `meow`.

É exibido sem newline automático.

O newline final lido de stdin é removido.

## `stash`

Append mutável:

```nyarn
scratch cats = ["Milo"]
stash cats, "Amora"
```

Regras:

- primeiro operando precisa resolver para clowder mutável;
- elemento precisa ser compatível;
- é statement;
- não produz valor útil na v1.

## `snatch`

Remove e retorna.

Último item:

```nyarn
mew last = snatch cats
```

Índice específico:

```nyarn
mew removed = snatch cats, 2
```

Índice negativo é permitido.

Lista vazia ou índice inválido → HISS.

Precisa de clowder mutável.

## `count`

Retorna `paw`.

```nyarn
purr count cats
purr count "mrrp"
```

Em strings, conta caracteres Unicode, não bytes.

## `sniffout`

Retorna `mood`.

Membership em clowder:

```nyarn
purr sniffout cats, "Amora"
```

Substring em string:

```nyarn
purr sniffout "Nyarn", "yar"
```

---

# 14. `maybe` e `emptybowl`

`emptybowl` só é aceito por `maybe T`.

Inválido:

```nyarn
mew nickname: meow = emptybowl
```

Válido:

```nyarn
mew nickname: maybe meow = emptybowl
```

## Narrowing

```nyarn
sniff nickname != emptybowl {
    # nickname: meow
    purr nickname
}
```

Inverso:

```nyarn
sniff nickname == emptybowl {
    purr "none"
}
nap {
    # nickname: meow
    purr nickname
}
```

Narrowing vale somente na branch onde a condição prova o estado.

`emptybowl` não é truthy/falsy.

Não sofre coerção implícita.

---

# 15. Conversões

Tipos base funcionam como funções de conversão.

## `paw(...)`

```nyarn
paw("17")
paw(17)
paw(17.0)
```

Regras:

- `paw(paw)` mantém valor;
- `paw(whisker)` só se finito e exatamente integral;
- `paw(meow)` parseia inteiro decimal válido;
- falha → HISS.

## `whisker(...)`

```nyarn
whisker(17)
whisker("1.72")
```

Regras:

- `whisker(paw)` promove;
- `whisker(whisker)` mantém;
- `whisker(meow)` parseia float válido.

## `meow(...)`

Retorna representação textual canônica de valores runtime suportados.

```nyarn
meow(42)
meow(purring)
```

## `mood(...)`

```nyarn
mood("purring")
mood("hissing")
```

Aceita exatamente essas strings para conversão textual.

Numeric → `mood` não existe.

`emptybowl` não pode ser convertido para tipo base não-optional.

---

# 16. Erros e HISS

## `hiss`

Lança erro runtime explícito:

```nyarn
sniff lives < 0 {
    hiss "cats cannot have negative lives"
}
```

Na v1, HISS não é capturável.

Sobe ao top-level e encerra execução do arquivo.

## Categorias

```text
HISS [syntax]
HISS [type]
HISS [runtime]
HISS [name]
HISS [io]
HISS [internal]
```

## Formato esperado

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

8 | sniff lives {
          ^^^^^

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

> [!important]
> **Useful first. Unhinged second.**
>
> A piada nunca substitui a informação técnica.

Um diagnóstico pode carregar:

- kind;
- arquivo;
- linha;
- coluna;
- source span;
- mensagem técnica;
- hint;
- mensagem felina;
- stack trace Nyarn.

## Sugestão de typo

```text
HISS [name] at main.nyarn:4:6
unknown identifier `nymae`

4 | purr nymae
         ^^^^^

did you mean `nyame`?

THE CAT FOUND NO SUCH THING UNDER THE COUCH.
```

## Stack trace

```text
HISS [runtime] at calc.nyarn:14:9
division by zero

  at divide()       calc.nyarn:14
  at calculate()    calc.nyarn:22
  at main_logic()   main.nyarn:8
  at <top-level>    main.nyarn:13

THE CAT HAS KNOCKED THE CALL STACK OFF THE TABLE.
```

## Erro interno

Bug do interpretador não deve parecer erro do usuário:

```text
NYARN INTERNAL ERROR

The interpreter tripped over its own yarn.
This is probably a Nyarn bug, not yours.
```

Raw Go panic deve ficar restrito a debug explícito, como `NYARN_DEBUG=1`.

---

# 17. CLI

Formas especificadas:

```bash
nyarn hello.nyarn
```

```bash
nyarn run hello.nyarn
```

Ambas executam o mesmo arquivo.

Rodar apenas:

```bash
nyarn
```

abre o REPL.

## Exit codes

```text
0  sucesso
1  runtime failure / hiss explícito
2  erro de syntax/parser
3  erro de type/checker
4  erro de CLI, arquivo ou I/O
```

Erros internos também usam código não-zero e precisam ser claramente rotulados como internos.

---

# 18. REPL

Exemplo:

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

Estado global persiste entre entradas bem-sucedidas.

Funções top-level declaradas no REPL também persistem.

## Multiline

```text
> pounce greet(nyame: meow) {
...     purr "Henlo, {nyame}"
... }
>
```

O REPL continua coletando input quando `{}`, `[]` ou `()` estão incompletos.

## Recuperação de erro

Erro de parse, check ou runtime não deve matar a sessão.

O estado precisa permanecer consistente após uma entrada falha.

---

# 19. Precedência de operadores

Da maior para a menor:

| # | Operadores |
|---:|---|
| 1 | calls e indexing |
| 2 | unary `nah`, `+`, `-` |
| 3 | `**` |
| 4 | `*`, `/`, `%` |
| 5 | `+`, `-` |
| 6 | `..` |
| 7 | `<`, `<=`, `>`, `>=` |
| 8 | `==`, `!=` |
| 9 | `andpaw` |
| 10 | `orpaw` |
| 11 | `=` |

`**` é right-associative.

`=` é right-associative.

Quando houver dúvida:

```text
use parentheses
```

O parser agradece.

## Assignment targets

Válidos:

- identificadores mutáveis;
- índices cujo root binding é mutável.

```nyarn
scratch lives = 9
lives = 8
```

```nyarn
scratch cats = ["Milo"]
cats[0] = "Luna"
```

Inválido:

```nyarn
(2 + 2) = 5
```

---

# 20. Runtime e modelo de execução

Esta seção é útil para entender como a linguagem se comporta e para quem está implementando a Nyarn.

## Valores runtime

O interpretador de referência precisa representar pelo menos:

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

`FunctionValue` e `RangeValue` não são keywords de tipo públicas.

## Ambientes

Estrutura conceitual:

```text
Global
  ↓
Function call
  ↓
Sniff / Hunt / outro bloco
```

Bindings armazenam, no mínimo:

```text
name
value
tipo declarado/inferido
mutable flag
source info
```

## Sinais de controle

Internamente:

```text
ReturnSignal
BreakSignal
ContinueSignal
HissSignal
```

Relacionados a:

```text
gift  -> ReturnSignal
leap  -> BreakSignal
prowl -> ContinueSignal
hiss  -> HissSignal
```

## Execução top-level

Depois de lexer/parser/checker passarem:

```text
Pass 1: registrar pounce top-level
Pass 2: executar statements top-level em ordem
```

É por isso que funções têm hoisting e bindings não.

## Ranges lazy

Ranges não devem precisar virar clowders gigantes na memória.

```nyarn
hunt i in 0..1000000 {
    # ...
}
```

pode iterar sobre um `RangeValue` lazy.

---

# 21. Coisas que NÃO existem na v1

Nyarn v1 deliberadamente não inclui:

- módulos/imports;
- implementação de `pspsps`;
- records/structs/classes;
- implementação de `breed`;
- nested functions;
- closures;
- default function arguments;
- slicing;
- custom range steps;
- operador de divisão inteira;
- erros capturáveis;
- `carefully` / `hissback` implementados;
- formatter / `nyarn fmt`;
- object methods;
- map/filter/reduce/sort/reverse como APIs ricas;
- operadores definidos pelo usuário;
- macros;
- generics além de `clowder T` e `maybe T`;
- package manager;
- LSP;
- bytecode VM;
- JIT.

> [!warning]
> “Seria legal” não significa “precisa entrar na v1”.
>
> O gato também precisa aprender YAGNI.

---

# 22. Exemplos completos

## 22.1 Hello Nyarn

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

## 22.2 Input + conversão

```nyarn
mew nyame = listen "Nyame: "
mew age_text = listen "Age: "
mew age: paw = paw(age_text)

purr "Henlo, {nyame}. You are {age}."
```

## 22.3 Clowder mutável

```nyarn
scratch cats: clowder meow = ["Milo", "Luna"]

stash cats, "Amora"
mew last = snatch cats

purr "removed: {last}"
purr "remaining cats: {count cats}"
```

## 22.4 Range + loop control

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

## 22.5 Recursão

```nyarn
pounce factorial(n: paw): paw {
    sniff n <= 1 {
        gift 1
    }

    gift n * factorial(n - 1)
}

purr factorial(5)
```

## 22.6 Globals mutáveis

```nyarn
scratch counter: paw = 0

pounce bump() {
    counter = counter + 1
}

bump()
bump()
bump()

purr counter
```

## 22.7 `maybe` + narrowing

```nyarn
mew nickname: maybe meow = emptybowl

sniff nickname == emptybowl {
    purr "No nickname. The bowl is empty."
}
nap {
    purr "Nickname: {nickname}"
}
```

## 22.8 Mini programa completo

```nyarn
pounce describe(cat: meow, lives: paw): meow {
    sniff lives <= 0 {
        gift "{cat} has become dramatically unavailable"
    }

    peek lives == 1 {
        gift "{cat} has exactly one bad decision left"
    }

    nap {
        gift "{cat} still has {lives} lives"
    }
}

mew nyame = listen "Nyame: "
scratch lives: paw = 3
scratch cats: clowder meow = [nyame]

stash cats, "Nyarn"
stash cats, "Milo"

hunt cat in cats {
    sniff cat == "Milo" {
        prowl
    }

    purr describe(cat, lives)
}

hunt lives > 0 {
    purr "life counter: {lives}"
    lives = lives - 1
}

sniff lives == 0 {
    purr "empty life inventory"
}
nap {
    hiss "THE LAWS OF FELINE PHYSICS HAVE FAILED"
}
```

---

# 23. Referência rápida

## Keywords

```text
mew       immutable binding
scratch   mutable binding

sniff     if
peek      else if
nap       else

hunt      loop
in        iteration relation
leap      break
prowl     continue

pounce    function
gift      return

purr      print
listen    input
hiss      raise runtime error

andpaw    and
orpaw     or
nah       not

stash     append
snatch    remove + return
count     length
sniffout  membership / substring
```

## Tipos

```text
paw       integer
whisker   float
meow      string
mood      boolean
clowder T homogeneous list
maybe T   optional
```

## Valores especiais

```text
purring    true
hissing    false
emptybowl  null / absence
```

## Operadores

```text
+  -  *  /  %  **
== != > >= < <=
=
..
```

## Lógica

```text
andpaw
orpaw
nah
```

## Reservado para depois

```text
pspsps
breed
carefully
hissback
```

---

# 🔗 Notas relacionadas

- [[Nyarn v1 - Introdução]]
- [[SPEC|Nyarn v1 — Especificação]]
- [[README|Nyarn — README]]

---

> [!success] Regra final
> Se esta nota disser uma coisa e a SPEC disser outra, **a SPEC vence**.
>
> A documentação pode ser bonita.
>
> A semântica precisa ser correta.

**Write code. Chase bugs. Hiss at type errors.**
