# site-generator-speed-01 — a faster drop-in for makefun-site 1.0.0

Entry for the make.fun bounty **site-generator-speed-01-run-20261122**
("Build a static site faster, 22 November 2026"). `solve.js` is a
dependency-free ES module whose `run(input)` returns the output tree;
`main.ts` is the harness driver (the baseline's driver, logic unchanged).

## Correctness first

The entry is a from-scratch reimplementation of makefun-site 1.0.0's
semantics, verified byte-identical to the reference on every practice case
(see `VERIFICATION.md`). Where the baseline README and its code differ, the
code governs — the entry was written against the reference implementation in
the kit, not against the README prose.

## What makes it faster

1. **Fused single-pass Markdown renderer.** The reference lexes into a token
   array and then renders it; this entry renders straight from the source
   lines. Inline markup is processed by a scanner that jumps from one markup
   character to the next with `indexOf` instead of stepping through every
   character in JS, and plain-text runs are escaped in bulk (one
   `escapeHtml` call per run instead of one regex call per character).
   Unescaped-closer searches (`**`, `]`, lone `*`) are `indexOf` plus a
   backslash-parity check, which is exactly equivalent to the reference's
   character-by-character escape-pair scan.
2. **Block dispatch on the first character.** The reference runs seven
   regexes on every line; this entry runs only the tests a line's first
   character can pass (blank lines are all spaces/tabs, fences start with
   `` ` ``, headings with `#`, rules with `-`/`*`, quotes with `>`, lists
   with `-`/`*`/digit). Same order, same outcomes.
3. **Compiled templates.** Each parsed template is compiled once per case
   into a JS function via `new Function` (the parse tree is unchanged, so
   parse errors stay identical; only render dispatch is compiled away).
   Helpers (`lookup`, `textOf`, `truthy`) stay shared and are passed in as
   arguments. If the platform refuses to compile, the entry falls back to
   the tree renderer — same output, slower (verified with
   `--disallow-code-generation-from-strings`: still 3/3 byte-identical).
4. **Cheap fast paths that are exact, not approximate.**
   - `slugify`: pure-ASCII strings skip the NFKD + `\p{M}` pass (NFKD is
     the identity on ASCII).
   - Front matter: the `\r` normalisation is skipped when the source has
     no `\r`.
   - Tag slugs are computed once per tag, not once per page that carries it.
   - `pageOf` reuses the post's card object instead of rebuilding it.
   - The template renderer appends into one array instead of concatenating
     a string per node.
5. **Same V8 tuning as the baseline.** `main.ts` sets
   `--no-incremental-marking` before importing `solve.js`, exactly as the
   house baseline does (the rules allow an entry to tune V8's flags too).

## What it does not do

- No caching between cases: every case is answered alone from its own
  input (`run` builds a fresh pipeline per call).
- No network calls, no file I/O, no WebAssembly, no native addon, no
  prebuilt binary. The only runtime code generation is `new Function`
  over source generated from the parsed template tree, with the tree
  renderer as fallback.
- No APIs newer than Node 22 (the VM here runs Node 24.22.1; the code uses
  nothing beyond Node 22 — `Object.hasOwn`, `??`, `?.`, `Map`/`Set`).

## Files

- `solve.js` — the entry (35,643 bytes).
- `main.ts` — the driver (baseline driver, logic identical).
- `package.json` — `{"type": "module"}` (avoids the reparse warning).
- `README.md` — this file.
- `VERIFICATION.md` — per-case diff results, timing numbers, and the
  95%-confidence assessment.

Total entry size ~37 KB, far under the 4 MiB repo limit. The `kit/` and
`work/` directories in this workspace are build/verification scratch, not
part of the entry.
