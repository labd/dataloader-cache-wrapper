---
"@labdigital/dataloader-cache-wrapper": patch
---

Build with tsdown instead of tsup, and validate the published package with
publint on every build.

The published output changes shape slightly as a result:

- The unused `iife` bundle is gone; only `esm` and `cjs` are emitted.
- `exports["."]` now declares `types` per condition, with `./dist/index.d.cts`
  for `require`. The single `index.d.ts` was interpreted as ESM under the
  `require` condition, so the types only resolved for CJS consumers that used a
  dynamic `import()`.
- `files` is set to `dist`, so the tarball no longer ships `src`, tests and
  build config. `sideEffects: false` is declared so bundlers can tree-shake.
- `@biomejs/biome` moves from `dependencies` to `devDependencies`; it is a
  build-time tool and was being installed by every consumer.
- `engines.node` is declared as `>=22`. Node 20 is end-of-life, and tsdown
  itself requires `^22.18.0 || >=24.11.0` -- on older versions it falls back to
  the `unrun` config loader, which is an optional peer dependency and not
  installed here. CI now validates on 22.x, 24.x and 26.x.
