# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Lawvere is a categorical programming language with effects, implemented in Haskell. The executable is named `bill`. Programs are point-free (no lambdas), inspired by category theory: values are arrows, datatypes are objects. Effects are modelled via free Freyd categories.

## Build / Run / Test

The project uses **hpack** — `package.yaml` is the source of truth; `lawvere.cabal` is generated (do not edit by hand, run `hpack`). It builds with either `stack` or `cabal`, and a Nix setup also exists.

- Build: `stack build` (or `cabal build`)
- Install `bill` to `~/.local/bin`: `stack install`
- Run the test suite: `stack test` (uses hspec-discover)
- Run a single hspec example: `stack test --ta '-m "<substring of it description>"'`
  (test descriptions look like `can run: examples/sum.law` — see [test/LawvereSpec.hs](test/LawvereSpec.hs))
- Format Haskell sources: `./scripts/format-hs.sh` (ormolu with custom extensions; runs `--check-idempotence`)
- Regenerate cabal from package.yaml: `hpack`
- Regenerate Nix derivation after dep changes: `cabal2nix . > nix/packages/lawvere.nix && direnv reload`
- ghcid loop for examples: `ghcid` reads [.ghcid](.ghcid) (runs `Main.dev` and re-triggers on example file changes)

GHC options are strict: `-Weverything -Werror` with a curated list of `-Wno-…` exceptions in [package.yaml](package.yaml). Treat new warnings as build failures.

### Running the language

```
bill <file.law>                  # run main arrow with the Haskell evaluator
bill -i [file.law]               # REPL (`:r` reload, `:q` quit)
bill --target js  <file.law>     # compile to JavaScript
bill --target vmcode <file.law>  # emit categorical-abstract-machine bytecode
bill --target vm     <file.law>  # run on the categorical abstract machine
bill --no-warnings -w …          # silence checker warnings
```

`.md` files are treated as literate Lawvere (fenced ` ```lawvere ` blocks); the project's own [README.md](README.md) is a runnable program and is part of the test suite.

## Architecture

Source tree:
- [src/Lawvere/](src/Lawvere/) — library; everything interesting lives here. [src/Lawvere.hs](src/Lawvere.hs) is an empty umbrella module.
- [bill/Main.hs](bill/Main.hs) — CLI driver and REPL (options-applicative + haskeline).
- [test/](test/) — hspec, discovered via `{-# OPTIONS_GHC -F -pgmF hspec-discover #-}` in [test/Spec.hs](test/Spec.hs).
- [examples/](examples/) — `.law` programs, also used as test fixtures.
- [js/lawvere.js](js/lawvere.js) — runtime shim emitted with the JS backend (declared in `data-files`).
- [nix/](nix/), [shell.nix](shell.nix), [nixkell.toml](nixkell.toml) — Nix build.

Module pipeline (rough flow inside `Lawvere.*`):

1. **Parse** — [Parse.hs](src/Lawvere/Parse.hs) (megaparsec). [Literate.hs](src/Lawvere/Literate.hs) extracts code blocks from Markdown via pandoc/commonmark. [File.hs](src/Lawvere/File.hs) ties them together (`parseFile` switches on `.md` suffix).
2. **AST** — [Core.hs](src/Lawvere/Core.hs) (identifiers, labels), [Scalar.hs](src/Lawvere/Scalar.hs), [Ob.hs](src/Lawvere/Ob.hs) (object/type expressions), [Expr.hs](src/Lawvere/Expr.hs) (arrow expressions), [Decl.hs](src/Lawvere/Decl.hs) (top-level: `DAr`, `DOb`, `DSketch`, `DInterp`, `DCategory`, `DEffCat`, `DEff`, `DEffInterp`), [Sketch.hs](src/Lawvere/Sketch.hs).
3. **Check** — [Check.hs](src/Lawvere/Check.hs) returns `(Either err _, warnings)` via `checkProg`. Type checker is incomplete (no row variables yet).
4. **Backends**, all consuming `[Decl]`:
   - Haskell evaluator: [Eval.hs](src/Lawvere/Eval.hs) — `evalMain`, with `Val = Rec | Tag | Sca | …` and `FreydDict` carrying effect interpretations (`~`, handlers, sum distributor).
   - Categorical abstract machine: [Instruction.hs](src/Lawvere/Instruction.hs) — `compileProg` / `runProg`.
   - JavaScript: `mkJS` (currently incomplete; only basic features).
   - `Comp.hs.disable` is intentionally disabled.
5. **Display** — [Disp.hs](src/Lawvere/Disp.hs) — pretty-printer used everywhere (`render`, `renderSimple`, `renderTerm`).

Top-level surface declarations to know about when touching parser/checker/eval together: `ar` (arrow), `ob` (object), `effect`, `category`, `effect_category`, `interpret`, `sketch`. Effect categories are introduced over a base category; running an effectful program requires mapping through an interpretation (see the State/Err example in [examples/partial-state.law](examples/partial-state.law) and the README).

## Conventions and gotchas

- `Protolude` is the prelude (`NoImplicitPrelude` is enabled globally). When importing from `base`, expect to qualify or import via Protolude.
- Many extensions are on by default — see the `default-extensions` list in [package.yaml](package.yaml) (notably `OverloadedLabels`, `DuplicateRecordFields`, `RecordWildCards`, `LambdaCase`, `TypeApplications`). `generic-lens` + `Data.Generics.Labels` are used heavily for field access (`use #names`, `#prog`, etc.).
- The test in [test/LawvereSpec.hs](test/LawvereSpec.hs) compares against exact rendered output strings; when changing the pretty-printer or evaluator semantics, expect to update these literals.
- `Paths_lawvere` (auto-generated) is used by the evaluator/tests to resolve `examples/*` and `js/lawvere.js` via `getDataFileName` — keep new runtime data files in the `data-files` list of [package.yaml](package.yaml).
- A GHC 9.8+ guard adds `-Wno-missing-role-annotations` (see the `when:` block at the bottom of [package.yaml](package.yaml)). The current resolver is `lts-24.41`.

## WebAssembly build (browser frontend)

A second executable `bill-web` ([bill-web/Main.hs](bill-web/Main.hs)) targets the GHC WebAssembly backend and powers a browser REPL. It's gated behind the `Web` cabal flag (off by default) and uses `GHC.Wasm.Prim` JSFFI to talk to xterm.js in the page. The frontend lives under [web/](web/) and uses a parchment / iron-gall colour palette — deliberately different from the sibling cpl project's dark VS Code theme.

Toolchain: the local GHC wasm distribution at `/Users/sakai/.ghc-wasm/` (GHC 9.14.1), separate from the native stack build's `lts-24.41` (GHC 9.10.3).

End-to-end local build:

```bash
source /Users/sakai/.ghc-wasm/env       # adds wasm32-wasi-{ghc,cabal}, node, etc. to PATH
./scripts/build-wasm.sh                  # → _site/lawvere.{wasm,js} + bundled web assets
./scripts/build-tutorial.sh              # → _site/tutorial.html (pandoc; requires pandoc)
python3 -m http.server -d _site 8000     # local preview at http://localhost:8000/
```

Build details:

- `scripts/generate-samples-js.sh` bundles `examples/*.law` and `README.md` into `web/samples.js` (gitignored; rebuilt each time). The in-browser `:l <path>` first looks here, falling back to a file picker when given no argument.
- `scripts/build-wasm.sh` runs `wasm32-wasi-cabal configure -fWeb`, builds `exe:bill-web`, then `post-link.mjs` (from `wasm32-wasi-ghc --print-libdir`) generates the JSFFI glue `lawvere.js`.
- With `-fWeb` on: the native `bill` executable and the test suite are marked `buildable: false`, and the library drops the `ansi-terminal` + `terminal-size` deps (CPP-guarded in [src/Lawvere/Disp.hs](src/Lawvere/Disp.hs)). The `--target js` REPL command throws in the wasm build ([src/Lawvere/Eval.hs](src/Lawvere/Eval.hs) `mkJS`).
- Browser files: [web/index.html](web/index.html), [web/lawvere-terminal.js](web/lawvere-terminal.js) (xterm.js + WASI shim from `@bjorn3/browser_wasi_shim`), [web/tutorial.css](web/tutorial.css), [web/manifest.json](web/manifest.json), [web/icon.svg](web/icon.svg) (favicons generated by the rsvg/magick snippet in [web/README.md](web/README.md)).
- Deployment: [.github/workflows/wasm-deploy.yaml](.github/workflows/wasm-deploy.yaml) installs `ghc-wasm-meta`, runs the two build scripts, and pushes `_site/` to GitHub Pages on push to `master`/`develop`.

When editing the library, remember that any module reachable from `bill-web` must also build under GHC 9.14.1 + wasm32-wasi. The native build still runs in CI via stack and the existing test suite; the wasm build is independent.
