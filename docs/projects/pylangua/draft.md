# PyLangua: Python in Polish

## Introduction

### Motivation

There is no user problem here. PyLangua starts from a personal itch: "what would Python look like written in Polish?"

I picked it on purpose. A self-imposed constraint forces real engineering decisions across parsing, tooling and language ecosystems - the kind of decisions that are hard to manufacture in a feature-sized side project.

### Goal

Take the constraint seriously enough to:

- Translate Python keywords, builtins, exceptions and dunders into Polish equivalents
- Build the surrounding tooling (transpiler, editor support, eventually a patched interpreter) so the result actually feels like writing Python
- Treat the project as a one-off learning artifact, not a maintained product

### Approach

Five stages, sequenced from cheapest proof of concept to deepest integration:

1. **Token-level transpiler** - write `.pypl`, transpile to `.py`, run it. Proof that the mapping works
2. **VS Code syntax highlighting** - so `.pypl` looks like real Python
3. **Patched CPython** - native Polish keywords, no transpilation step
4. **Editor language support** (LSP-shaped) - completion, diagnostics
5. **Generalisation** - if mapping-driven tooling works for Polish, it works for any language

Stages 1 and 2 are done. The post focuses on those, with a roadmap for the rest.

## Implementation

### Stage 1: Token-level transpiler

Source repo: [`pylangua-transpiler`](https://github.com/PanstwowyZakladKompilacji/pylangua-transpiler).

The transpiler is intentionally kept small to be able to deliver a demo of what this is about. It tokenises `.pypl` source with the standard library `tokenize` module, replaces any `NAME` token whose string matches a Polish-to-English entry, and writes the result to `.py`. The CLI then executes that `.py` file with the current Python interpreter.

```python
for tok in tokenize.generate_tokens(StringIO(code).readline):
    if tok.type == tokenize.NAME and tok.string in mapping:
        result.append(mapping[tok.string])
    else:
        result.append(tok.string)
```

That is the entire transformation. No AST walk, no parser, no schema.

### Stage 2: VS Code syntax highlighting

Source repo: [`pylangua-syntax`](https://github.com/PanstwowyZakladKompilacji/pylangua-syntax).

A vendored MagicPython TextMate grammar is regenerated from JSON mappings via a stdlib-only Python script. The mapping files (`mappings/pypl/*.json`) are the single source of truth: the same JSON drives the transpiler, the editor highlighter, and any future patched interpreter.

Three Polish keywords are dangerous in a regex grammar: `z` (from), `w` (in), `i` (and). They collide with everyday letters and would false-positive across normal code. The merge script handles them in narrow contexts only (import rule, `for ... in ...` generator) and excludes them from broad operator and illegal-name patterns.

### Example

```python
importuj math
z math importuj pi jako π

klasa KlasaBazowa(obiekt):
    """przykład"""

    zdefiniuj __inicjalizuj__(ja, x):
        ja.x = x

    zdefiniuj metoda(ja):
        jeśli ja.x:
            zwróć Prawda
        zwróć Fałsz


zdefiniuj główna():
    spróbuj:
        wypisz(wartość_bezwzględna(-1))
    przechwyć_wyjątek Wyjątek jako e:
        zgłoś_wyjątek BłądWartości("błąd") z e
    na_koniec:
        nic_nie_rób


dla i w zakres(3):
    oczekuj główna()
```

### Key design decisions

1. **Mappings as data, not code**

    JSON mapping files are the single source of truth. The transpiler, the editor grammar, and any future patch consume the same data. Adding or changing a translation is a one-line edit in JSON.

2. **Token-level instead of AST-level transpilation**

    `tokenize` is enough for a one-to-one identifier rewrite. An AST pass would be more "correct" but would also need to reconstruct source for execution, and would not buy anything for this scope.

3. **Two repositories, not one monorepo**

    The grammar work is TextMate-shaped; the transpiler is pure Python. Splitting them keeps each repo focused and lets the editor extension be packaged independently.

## Roadmap

Stages 3-5 are not built. They are described here as decisions, not promises.

### Stage 3: Patched CPython - shim vs full rename

The central design choice for stage 3 is how deep the rename goes.

**Option A - shimmed builtins.** Patch only the grammar and lexer for Polish keywords and literals. Builtins, exceptions and dunders are aliased at runtime via a small module loaded from `Lib/site.py`. English names still resolve, so PyPI packages keep working unchanged. Effort: bounded. Interop: full. Trade-off: the rename is cosmetic past the parser; the stdlib is still English under the hood.

**Option B - full rename.** Patch the grammar and rename Polish identifiers all the way through CPython, including stdlib source. No English fallback. Effort: large. Interop with PyPI: broken. Trade-off: the rename is real all the way down; the cost is permanent divergence from upstream.

I am committing to Option B. The point of PyLangua is not interop - it is to see the rename through the entire stack. Shimming would be the right answer for a maintained product; full rename is the right answer for a learning artifact whose value is precisely the depth of the rewrite.

Stage 3 is "done" when:

- Polish keywords parse natively
- Polish builtins, exceptions and dunders resolve without a runtime alias layer
- Tracebacks contain Polish names natively
- The result is installable as a venv interpreter

### Stage 4: Editor language support

Out of scope for now. Future direction: a lightweight LSP that proxies Pylance and remaps names on the way in and out. Not committed.

### Stage 5: Generalisation

Only worth doing if at least one external contributor expresses interest after stages 1-3. Until then, treating PyLangua as a single-language artifact keeps the scope honest.

## Trade-offs and Risks

- **Scope.** Five stages is large for a side project. Stages 3-5 may stall; this post commits only to stages 1 and 2 as shipped artifacts.
- **Maintenance posture.** Explicit non-goal: I am not maintaining PyLangua against future Python releases. It is frozen against Python 3.11 and a specific MagicPython grammar snapshot. Treating it as a frozen artifact is what makes the scope honest.
- **Identifier collisions.** The Polish words `i`, `w`, `z` collide with common identifiers and one-letter variables. The grammar handles them in narrow contexts; the transpiler is naive and will mistranslate them as bare identifiers (the bundled test sample uses `im` instead of `i` to sidestep this).
- **Tooling fragmentation.** Two repos, no shared CI yet, no tests in the transpiler. Acceptable for the current stage; called out so the gap is visible.
- **Diacritics.** `ł`, `ą`, `ę` and friends are valid Python identifier characters but can be friction-prone in some toolchains. Worth flagging up front.

## Reflection

### Takeaways

- **Most of a language is tooling, not syntax.** A working transpiler is a couple of dozen lines. Making it feel like a real language - editor support, runtime semantics, error messages, distribution - is where the actual work lives.
- **Pick a silly constraint to learn a serious system.** "Python in Polish" is a deliberately narrow framing. The narrowness is what made it useful: it forced a path through `tokenize`, TextMate grammars, CPython internals and language-server territory in a way a feature project would not.
- **Naming the non-goal is part of the design.** Saying "this is a frozen one-off" up front is what lets the project be honest about its scope without losing credibility.

### What this is, and what it is not

PyLangua is a portfolio piece. It is not a product, not a community tool, and not a serious proposal for Polish-speaking developers. It is a deliberately-scoped exercise in shipping increments through a multi-stage technical project under a self-imposed constraint.

## Links

[Transpiler repo](https://github.com/PanstwowyZakladKompilacji/pylangua-transpiler) (stage 1)<br>
[VS Code extension repo](https://github.com/PanstwowyZakladKompilacji/pylangua-syntax) (stage 2)
