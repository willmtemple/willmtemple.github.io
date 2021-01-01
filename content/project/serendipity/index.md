---
title: Serendipity
subtitle: Educational development environment with new ideas about blocks-based programming
summary: A programming language and editor for learning and exploration.
tags:
- Education
- Programming Languages
- Blocks

# Optional external URL for project (replaces project detail page).
external_link: ""

date: 2020-12-01

image:
  caption: Prototype Serendipity Editor Interface
  focal_point: Smart
url_code: "https://github.com/willmtemple/serendipity"
url_pdf: ""
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""

highlight_languages: ['lisp', 'common-lisp']
---

**Serendipity** is a new prototype programming environment for desktops and the
web. Currently, it is in the proof-of-concept stage. It is powered by a tiny
core language that is:

- purely-functional.
- dynamically-typed.
- lazy-evaluated.

The design and overall philosophy of Serendipity is inspired by the many
wonderful programming languages that I have had the pleasure to use, primarily
[Racket][racket], JavaScript, Rust, and Clojure. My greatest aspiration for it
is that students and experienced programmers alike might use it to discover
powerful and fundamental ideas in Computer Science through exploring an
interactive programming environment.

I designed Serendipity with the intent of developing several front-ends. If I
succeed, then it should be easy to develop completely new front-ends for
serendipity that use a variety of editing modalities. Currently, there are two
front-ends: the basic blocks-based editor featured in the screenshot above and
a Lisp-like front-end that I use for debugging. The blocks editor uses React.js
and MobX state management. The UI is powered by ordinary SVG.

Currently, Serendipity is implemented entirely in TypeScript, though I intend
to rewrite some or all of it in Rust (or hopefully in a Serendipity front-end)
as needed.

## Compared to Other Block Programming Engines

Serendipity's block editor was borne out of frustrations with existing block
programming toolkits, in particular Google's [Blockly][blockly]. The
fundamental problem with Blockly and its many forks is that it treats blocks as
merely a representation of text. When a blockly program is compiled, it is
transformed into text, and then the text is processed using conventional
compilers/runtimes. While this has some advantages, it introduces a lot of
complexity in the process of returning useful diagnostic information to the
user in the block workspace. It also means that, in some cases, text programs
cannot be represented by blocks (try writing a higher-order program in a
Blockly-based language!).

Serendipity's editor instead views blocks as a direct representation of the
program's _abstract syntax tree_. Serendipity's editor is therefore a
_structure editor_, and when a user manipulates the blocks, they manipulate the
abstract syntax tree directly. This means that there is no parser in the web
editor, as the principal data structure of the program is the syntax tree,
rather than a text file as would be the case with conventional text-based or
Blockly-based programming.

## Compiler

Serendipity's compiler, though simple, contains some unusual transformations
that the compiler-interested may find intriguing. While the core runtime
language is purely-functional, the front-end languages allow for procedural
programming. For example, consider the following iterative program (using the
Lisp front-end):

```lisp
; A sequence of integers starting at n
(define (seq n) (cons n (seq (+ n 1))))

(main [proc
  (for-in [i (seq 0)] (do [proc
    (if (>= i 10) (break))
    (print i)
  ]))
])
```

In this front-end, `proc` is a fundamental constructor for a procedural block
of code. While this construction appears similar to `begin` in Scheme (by
design), its implementation is much different. When this program is compiled,
Serendipity will apply a series of transformation that reduce procedural
programs into functional programs using a continuation-passing style with an
explicit "world" argument.

In the language of Serendipity's runtime, the `main` program above is reduced
to a pure functional computation (formatted and annotated for "readability"):

```text
;; Main entry point, with some shim code for bootstrapping CPS
;; __k defines 
((λ__k.λ__w.(
 ;; The fundamental looping constructor provides a continuation for breaking
 λ__brk.(
  ;; Iteration recursively binds a single iteration as a continuation until
  ;; the loop is broken, either by an explicit `(break)` or by the iterator
  ;; returning nil. (`seq` below is bound in the environment.)
  λ__loop.((__loop __w) (seq 0)) (
   ;; Fixed-point combinator allows for recursive binding of the loop
   ;; body
   λf.(λx.(x x) λx.(f (x x))) λ__loop.λ__w.λ__iter.(
    ;; Test for iterator termination
    (== __iter ∅)
    ;; If the iterator terminated (produced nil), we call the break
    ;; continuation
    ? (__brk __w)
    ;; Otherwise, run the iterator body, which starts with an (if) construction
    : (λi.(λ__w.((λ__k.λ__w.(λ__next.(
     ;; Do you recognize this? It's just the test for (>= i 10) that appeared
     ;; in the base program. Pure expressions are reproduced verbatim.
     (>= i 10)
     ;; The explicit call to break is rendered as an immediate call to the
     ;; breaking continuation 
     ? (__brk __w)
     ;; Otherwise, evaluate the __next continuation (specifies the continuation
     ;; that handles the result of the entire (if) construction)
     : (__next __w))
     ;; The next continuation calls the `print` core builtin function.
     ;; Eventually this will be modeled as a monadic structure. For now it
     ;; simply returns the world as-is
     λ__w.(
       (λ__k.λ__w.((__core.print i) ((λ__k.λ__w.(__k __w) __k) __w)) __k)
       __w))
     ;; Evaluate __iter[1] (next value in the iterator) and loop over it
     ;; by finally cascading the world arguments and continuations
     λ__w.(λ__w.((__loop __w) __iter[1]) __w)) __w) __w) __iter[0])
   )))
   ;; This is bootstrapping code, though you can see that the "world" is
   ;; initialized to nil in the final argument (keep in mind that this is
   ;; reversed, so nil is actually the *first* argument evaluated and passed
   ;; into this CPS-call expression).
   λ__w.((λ__k.λ__w.(__k __w) __k) __w)) λ__w.__w) ∅)
```

The choice to use this extremely simple Church-ish language, though not very
readable (that said, I don't think it is that much worse than assembly
language), enables all of the theoretical static analysis and optimization that
is only useful in pure-functional programs (to summarize: it is very easy to
statically analyze a pure-functional program). I believe that this pattern of
continuation-passing combined with dead-code and tail-call elimination along
with the essence of laziness (you'll find that _all_ of the function calls
above are tail recursive except for the infinite, where we are assisted by
laziness) provides a very promising substrate for building Serendipity far into
the future.

## What's Next?

In addition to adding enough features to the editor to take Serendipity from a
proof-of-concept stage to a usable Alpha stage (currently, you can't save a
program and have to use the debugging console to spawn new blocks), I am
developing a non-Lisp-like textual representation of Serendipity programs that
will have two front-ends: a text parser and a text-like structure editor
(similar to the incredibly powerful editor of the [Hazel][hazel] project) as a
counterpart to the blocks-based structure editor.

Eventually, I will rewrite most of the block rendering code to use either HTML
`canvas` or a GPU subsystem (WebGL or WebGPU), as I am reaching the limits of
what I can easily accomplish within React and SVG due to uncontrollable
accumulations of SVG jank.

Beyond the editor, the next monster problem to solve is to define and implement
a type system. I have become completely enamored with TypeScript's approach to
type-checking JavaScript: interface-driven optional typing. That said,
TypeScript isn't perfect, and there are mistakes that were made along its
development into the beloved language that it is today, and I now have the
benefit of being able to learn from those mistakes. I am attracted to the idea
of truly optional typing, in which a user _might_ provide a type (but doesn't
have to), and we can check it statically, with no runtime cost, if a type is
provided.

However, implementing a type checker isn't the reason it's a monster problem.
Writing some software for basic optional typing is simple enough. The
monstrosity comes from trying to figure out how to represent optional type
annotations in a blocks-based programming language. None of the block
environments that I'm familiar with do this very satisfactorially (or at all),
so if you are aware of any, send me an email or leave a comment below!

[blockly]: https://developers.google.com/blockly/
[hazel]: https://hazel.org/
[racket]:  https://racket-lang.org/

