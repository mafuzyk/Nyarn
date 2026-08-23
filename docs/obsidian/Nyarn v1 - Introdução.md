---
title: Nyarn v1 — Introdução
aliases:
  - Introdução ao Nyarn
  - Nyarn 101
  - Começando com Nyarn
tags:
  - nyarn
  - programming-language
  - guide
  - introduction
status: design-aprovado
version: v1
language: pt-BR
cssclasses:
  - nyarn-doc
---

# 🐈‍⬛ Nyarn v1 — Introdução

> [!quote] Nyarn, em uma frase
> **Nyarn é o que acontece quando alguém dá acesso a design de linguagens para um gato.**

> [!important] Estado atual
> A **linguagem Nyarn v1 já tem design e especificação aprovados**, enquanto o interpretador de referência em Go ainda está sendo implementado.
>
> Esta nota ensina **como a Nyarn v1 foi projetada para funcionar**. Se você quiser regras formais, consulte [[SPEC|Nyarn v1 — Especificação]]. Para uma referência muito mais completa e consultável, use [[Nyarn v1 - Guia Completo]].

---

## 🧶 O que é Nyarn?

Nyarn é uma linguagem de programação pequena, interpretada e propositalmente felina.

Só que a piada termina no vocabulário.

Por baixo de palavras como `mew`, `purr`, `paw` e `emptybowl`, existe uma linguagem com:

- lexer próprio;
- parser próprio;
- AST própria;
- tipagem opcional;
- inferência de tipos;
- checker semântico separado do runtime;
- escopo léxico;
- funções e recursão;
- imutabilidade por padrão;
- listas homogêneas;
- erros estruturados;
- stack trace Nyarn;
- interpretador tree-walk em Go.

> [!summary] Filosofia
> **A Nyarn pode ser unserious. A semântica não.**
>
> O objetivo é que a linguagem seja divertida de escrever, mas previsível de aprender, implementar e depurar.

---

# 1. Seu primeiro programa

Arquivos Nyarn usam a extensão:

```text
.nyarn
```

Um Hello World canônico pode ser:

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

Lendo linha por linha:

1. `mew nyame = "Nyarn"` cria uma variável imutável.
2. `mew happy: mood = purring` cria um valor booleano tipado.
3. `sniff happy` significa “se `happy` for verdadeiro”.
4. `purr` imprime uma expressão.
5. `nap` é o `else`.

Se traduzíssemos a ideia para palavras tradicionais:

```text
let nyame = "Nyarn"
let happy: bool = true

if happy {
    print("Henlo from Nyarn!")
} else {
    print("zzz...")
}
```

A estrutura é familiar. O gato só se recusou a usar nomes normais.

---

# 2. O vocabulário essencial

Antes de qualquer detalhe, memorize só isto:

| Nyarn | Significado |
|---|---|
| `mew` | declarar binding imutável |
| `scratch` | declarar binding mutável |
| `purr` | imprimir |
| `listen` | ler entrada |
| `sniff` | if |
| `peek` | else if |
| `nap` | else |
| `hunt` | loop |
| `leap` | break |
| `prowl` | continue |
| `pounce` | declarar função |
| `gift` | return |
| `hiss` | lançar erro |

E os tipos principais:

| Nyarn | Tipo tradicional |
|---|---|
| `paw` | inteiro |
| `whisker` | float |
| `meow` | string |
| `mood` | boolean |
| `clowder T` | lista homogênea de `T` |
| `maybe T` | valor opcional |

Valores especiais:

```nyarn
purring    # true
hissing    # false
emptybowl  # null / ausência
```

> [!fun] Prioridades bem definidas
> `emptybowl` existir como equivalente a `null` foi uma decisão de linguagem surpreendentemente importante.
>
> Também é, objetivamente, uma tragédia felina.

---

# 3. `mew` e `scratch`

A Nyarn prefere imutabilidade.

## `mew`

```nyarn
mew nyame = "Mafu"
```

Depois disso:

```nyarn
nyame = "Nyarn"
```

é erro.

`mew` significa: **esse binding não deve mudar**.

## `scratch`

Se você quer mutação, precisa declarar a intenção:

```nyarn
scratch lives = 9
lives = 8
```

### Com listas

```nyarn
mew cats = ["Milo", "Luna"]
cats[0] = "Dexter"  # HISS
```

Mas:

```nyarn
scratch cats = ["Milo", "Luna"]
cats[0] = "Dexter"
```

é válido.

> [!tip]
> Se você não sabe se algo precisa ser mutável, comece com `mew`.
>
> Use `scratch` quando houver uma razão real para alterar o valor.

---

# 4. Tipagem sem sofrimento

Nyarn possui **tipagem opcional**.

Você pode escrever:

```nyarn
mew age = 17
mew height = 1.72
mew nyame = "Mafu"
mew sleepy = purring
```

A linguagem infere:

```text
17         -> paw
1.72       -> whisker
"Mafu"     -> meow
purring    -> mood
```

Ou pode ser explícita:

```nyarn
mew age: paw = 17
mew height: whisker = 1.72
mew nyame: meow = "Mafu"
mew sleepy: mood = purring
```

Se você escreveu o tipo, ele vira contrato:

```nyarn
mew age: paw = "seventeen"
```

→ **HISS de tipo.**

A Nyarn não vai fingir que uma string é um inteiro só porque seria conveniente.

---

# 5. `mood`: booleanos felinos

O tipo booleano é:

```text
mood
```

E seus valores são:

```nyarn
purring  # true
hissing  # false
```

Exemplo:

```nyarn
mew hungry: mood = purring
mew angry: mood = hissing
```

## Não existe truthiness geral

Isto é válido:

```nyarn
sniff hungry {
    purr "FOOD"
}
```

Isto não:

```nyarn
mew lives = 9

sniff lives {
    purr "???"
}
```

Condições precisam ser `mood`.

> [!warning] HISS
> `paw`, `meow`, `clowder` e `emptybowl` não são automaticamente “truthy” ou “falsy”.
>
> **THE CAT REFUSES TO GUESS WHAT YOU MEANT.**

Operadores lógicos:

```text
andpaw  -> and
orpaw   -> or
nah     -> not
```

```nyarn
sniff hungry andpaw nah sleepy {
    purr "time to destroy the kitchen"
}
```

---

# 6. Condições: `sniff`, `peek`, `nap`

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

Mapa mental:

```text
sniff -> if
peek  -> else if
nap   -> else
```

Você pode ter vários `peek` e no máximo um `nap` final.

Cada bloco `{ ... }` cria um novo escopo.

```nyarn
mew nyame = "Mafu"

sniff purring {
    mew nyame = "Nyarn"
    purr nyame  # Nyarn
}

purr nyame      # Mafu
```

Shadowing entre escopos é permitido.

Redeclarar o mesmo nome dentro do mesmo escopo não é.

---

# 7. Loops: `hunt`

`hunt` cobre dois casos.

## Loop condicional

```nyarn
scratch lives = 3

hunt lives > 0 {
    purr "{lives} lives left"
    lives = lives - 1
}
```

## Iteração

```nyarn
mew cats = ["Milo", "Luna", "Dexter"]

hunt cat in cats {
    purr cat
}
```

Controle de loop:

```text
leap   -> break
prowl  -> continue
```

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

A variável criada por `hunt cat in cats` é imutável.

---

# 8. Ranges

Ranges são half-open:

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

Ou seja:

```text
0..10
```

representa 0 até 9.

Na v1:

- limites de range são `paw`;
- não existe step customizado;
- o runtime deve tratá-los de forma lazy.

---

# 9. Funções: `pounce`

```nyarn
pounce add(a: paw, b: paw): paw {
    gift a + b
}

purr add(2, 3)
```

- `pounce` declara função;
- `gift` retorna valor.

Tipos de parâmetros e retorno são opcionais:

```nyarn
pounce greet(nyame) {
    purr "Henlo, {nyame}"
}
```

## Retorno vazio

```nyarn
pounce stop() {
    gift
}
```

retorna `emptybowl`.

Chegar ao fim da função sem `gift` também retorna `emptybowl`.

## Regras da v1

Funções:

- só podem ser declaradas no top-level;
- têm hoisting;
- aceitam recursão;
- exigem quantidade exata de argumentos;
- não têm argumentos default;
- não podem ser aninhadas;
- não têm closures.

Por causa do hoisting, isto funciona:

```nyarn
greet("Mafu")

pounce greet(nyame: meow) {
    purr "Henlo, {nyame}"
}
```

Variáveis comuns **não** são hoisted.

---

# 10. Clowders

`clowder` é a lista da Nyarn.

```nyarn
mew cats: clowder meow = ["Milo", "Luna", "Dexter"]
```

Clowders são homogêneas.

```nyarn
mew numbers = [1, 2, 3]
```

→ `clowder paw`

Isto é inválido:

```nyarn
mew chaos = [1, "cat", purring]
```

## Lista vazia

```nyarn
mew cats = []
```

não possui informação suficiente para inferir tipo.

Use:

```nyarn
mew cats: clowder meow = []
```

## Índices

```nyarn
purr cats[0]
purr cats[-1]
```

Índice negativo funciona como em Python.

Índice inválido gera HISS.

Não retorna `emptybowl` silenciosamente.

---

# 11. Operações de clowder

```nyarn
stash cats, "Amora"
```

adiciona ao final.

```nyarn
mew last = snatch cats
```

remove e retorna o último item.

```nyarn
mew removed = snatch cats, 1
```

remove por índice.

```nyarn
purr count cats
```

retorna tamanho.

```nyarn
purr sniffout cats, "Amora"
```

retorna `purring` ou `hissing` para membership.

Operações mutáveis como `stash` e `snatch` exigem uma clowder alcançada por `scratch`.

---

# 12. Strings

Strings usam aspas duplas:

```nyarn
mew nyame = "Mafu"
```

Escapes básicos:

```text
\n
\t
\"
\\
```

## Interpolação

```nyarn
mew lives = 9

purr "{nyame} has {lives} lives"
purr "next life: {lives - 1}"
```

A interpolação aceita expressões.

Chaves literais:

```nyarn
purr "{{hello}}"
```

→ `{hello}`

## Sem coerção mágica

```nyarn
purr "lives: " + 9
```

é erro.

Use:

```nyarn
purr "lives: {9}"
```

ou:

```nyarn
purr "lives: " + meow(9)
```

---

# 13. Entrada e conversões

```nyarn
mew nyame = listen "Nyame: "
```

`listen` retorna `meow`.

Sem prompt:

```nyarn
mew input = listen
```

Conversões explícitas usam o próprio nome do tipo:

```nyarn
paw("17")
whisker("1.72")
meow(42)
mood("purring")
```

Exemplo real:

```nyarn
mew age_text = listen "Age: "
mew age: paw = paw(age_text)
```

> [!note]
> Nyarn prefere conversão explícita a “adivinhar” que `"5"` provavelmente queria dizer `5`.

---

# 14. `maybe` e `emptybowl`

Tipos normais não aceitam `emptybowl`.

```nyarn
mew nickname: meow = emptybowl
```

→ HISS.

Declare possibilidade de ausência:

```nyarn
mew nickname: maybe meow = emptybowl
```

## Narrowing

```nyarn
sniff nickname != emptybowl {
    # nickname é meow aqui
    purr "Henlo, {nickname}"
}
```

E o inverso:

```nyarn
sniff nickname == emptybowl {
    purr "No nickname. Tragic."
}
nap {
    # nickname é meow aqui
    purr nickname
}
```

Isso mantém ausência explícita sem espalhar nullability invisível por todo o programa.

---

# 15. Números e promoção

Operadores:

```text
+  -  *  /  %  **
```

Comparações:

```text
==  !=  >  >=  <  <=
```

Existe exatamente uma promoção numérica implícita:

```text
paw -> whisker
```

```nyarn
mew a: paw = 5
mew b: whisker = 2.5
purr a + b
```

→ `whisker`

Divisão sempre retorna `whisker`:

```nyarn
purr 5 / 2
# 2.5
```

Potência é associativa à direita:

```nyarn
2 ** 3 ** 2
```

significa:

```text
2 ** (3 ** 2)
```

Comparações numéricas usam a mesma promoção:

```nyarn
5 == 5.0  # purring
```

Mas:

```nyarn
5 == "5"
```

é erro de tipo.

---

# 16. `hiss`: falhar explicitamente

```nyarn
sniff lives < 0 {
    hiss "cats cannot have negative lives"
}
```

Na v1, `hiss` não pode ser capturado.

Ele sobe até o top-level e encerra a execução com status diferente de zero.

`carefully` e `hissback` estão reservados para uma versão futura.

---

# 17. Diagnósticos

A regra sagrada é:

> **Useful first. Unhinged second.**

Exemplo:

```text
HISS [type] at main.nyarn:8:12
expected mood, got paw

8 | sniff lives {
          ^^^^^

THAT IS NOT A MOOD!!!
THE CAT REFUSES TO GUESS WHAT YOU MEANT.
```

Primeiro vem:

- categoria;
- arquivo;
- linha e coluna;
- source span;
- mensagem técnica;
- hint, quando útil;
- stack trace, em runtime errors.

Depois o gato pode perder completamente a compostura.

Exemplo de recursão profunda:

```text
HISS [runtime]
maximum call depth exceeded

THE YARN IS TOO TANGLED!!!
IS THE PROGRAM RECURSING?!
CAN'T KEEP UP!!!
```

---

# 18. Um programa um pouco maior

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

Se você consegue ler esse programa sem precisar decodificar cada keyword, você já entendeu a maior parte do Nyarn usado no dia a dia.

---

# 19. O que fica fora da v1

Nyarn v1 não inclui:

- imports/módulos;
- implementação de `pspsps`;
- classes;
- records/structs;
- implementação de `breed`;
- funções aninhadas;
- closures;
- parâmetros default;
- slicing;
- range steps customizados;
- divisão inteira dedicada;
- erros capturáveis;
- `carefully` / `hissback`;
- formatter;
- LSP;
- package manager;
- bytecode VM;
- JIT.

> [!quote]
> Às vezes, design de linguagem também significa saber quando parar de adicionar coisa.

---

# 20. Para onde ir agora

- [[Nyarn v1 - Guia Completo]] — referência completa da linguagem.
- [[SPEC|Nyarn v1 — Especificação]] — fonte normativa.
- [[README|Nyarn — README]] — visão pública e unserious do projeto.

---

# 🧷 Cheat sheet

```text
DECLARAÇÃO
mew       imutável
scratch   mutável

TIPOS
paw       int
whisker   float
meow      string
mood      bool
clowder   lista homogênea
maybe     opcional

VALORES
purring   true
hissing   false
emptybowl ausência

CONDIÇÕES
sniff     if
peek      else if
nap       else

LÓGICA
andpaw    and
orpaw     or
nah       not

LOOPS
hunt      loop
leap      break
prowl     continue

FUNÇÕES
pounce    function
gift      return

I/O E ERROS
purr      print
listen    input
hiss      throw/fail

CLOWDER
stash     append
snatch    remove + return
count     length
sniffout  contains
```

> [!success] Você terminou a introdução
> Agora você já sabe Nyarn o bastante para ler e escrever programas pequenos.
>
> O próximo passo natural é [[Nyarn v1 - Guia Completo]].

---

**Write code. Chase bugs. Hiss at type errors.**
