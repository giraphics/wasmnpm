[![Build](https://github.com/giraphics/wasmnpm/actions/workflows/build.yml/badge.svg)](https://github.com/giraphics/wasmnpm/actions/workflows/build.yml)

# wasmnpm — C++ to WebAssembly, packaged for npm

An unmodified checkout of [TheLartians/modern-wasm-starter](https://github.com/TheLartians/modern-wasm-starter),
kept here as a reference for the toolchain it demonstrates: type-safe C++
compiled to WebAssembly and consumed from TypeScript, with the `.d.ts`
declarations **generated from the C++** rather than written by hand.

There is nothing to look at — this is a library, not an application, so this
README has no screenshots.

## What the toolchain does

The C++ in `wasm/source/` declares a `Greeter` class through
[Glue](https://github.com/TheLartians/Glue). At build time Emscripten compiles
it and Glue emits `source/WasmModule.js` plus `source/WasmModule.d.ts`, so the
TypeScript wrapper in `source/wasmWrapper.ts` is type-checked against the real
C++ surface:

```ts
await withGreeter((greeterModule) => {
  const greeter = new greeterModule.Greeter("Wasm")
  greeter.greet(greeterModule.LanguageCode.EN)   // "Hello, Wasm!"
})
```

Those generated files are build artifacts and are not in the repository, which
is why `npx tsc` on a fresh clone reports `Cannot find module './WasmModule'`.
That is expected — build the WASM first.

Memory is handled with scopes (`withGreeterScope`, `persistGreeterValue`,
`deleteGreeterValue`), since C++ objects crossing into JS have no garbage
collector to reclaim them.

```text
wasm/source/main.cpp        emscripten entry point; registers the Glue module
wasm/source/wasmGlue.cpp    the C++ surface exposed to JS
wasm/CMakeLists.txt         CMake + CPM.cmake dependency fetching
source/wasmWrapper.ts       the typed JS wrapper and scope handling
source/index.ts             public API, re-exported under Greeter* names
__tests__/wasm.ts           jest tests that call across the boundary
```

## Requirements

**Emscripten is mandatory.** Nothing here builds without it — not even
`npm install`, because the `prepare` script compiles the WASM. Install and
activate the [emsdk](https://emscripten.org/docs/getting_started/downloads.html)
first, plus CMake.

```bash
npm install                 # runs prepare: configure + build wasm, then tsc
npm test                    # jest, against the built module
npm run build:wasm          # rebuild C++ only
npm run check:style         # prettier + Format.cmake
```

Without emsdk on the PATH, `npm install` fails at the `emcmake` step. Use
`npm install --ignore-scripts` if you only want the JS dependencies.

## State of this checkout

Verified on this machine (macOS, Node 22, no emsdk installed):

- `npm install` **failed** — the `prepare` script called `yarn run
  configure:wasm` while every other call in the same line used npm, so it died
  with `sh: yarn: command not found` before ever reaching Emscripten. That is
  now `npm run configure:wasm`; the failure is at least an honest emsdk error.
- `npm install --ignore-scripts` succeeds.
- `npx tsc` reports missing `./WasmModule` — expected without a WASM build, as
  above.
- The WASM build itself is **unverified here**; there is no emsdk on this
  machine. CI installs one, so that is where it gets exercised.

## Workflows

- [`build.yml`](.github/workflows/build.yml) — installs emsdk 2.0.31, builds,
  tests, and checks formatting on push and pull request. It tried to install
  `clang-format` with **`brew` on an `ubuntu-latest` runner**, where brew does
  not exist; that step now uses `apt-get`. `actions/checkout` and
  `actions/cache` were pinned at v2, which GitHub has since retired, and are
  now v4.
- [`publish.yml`](.github/workflows/publish.yml) — published to npm on every
  push to master. `package.json` still carries the upstream package name and
  author, so that would have aimed at **someone else's npm package**. The
  automatic trigger is disabled; it is `workflow_dispatch` only. Before
  enabling it, change `name` and `author` in `package.json` and set
  `NPM_AUTH_TOKEN`.

## Credits

[modern-wasm-starter](https://github.com/TheLartians/modern-wasm-starter) by
Lars Melchior, under the license in [`LICENSE`](LICENSE). The upstream README
documents the template's design in more detail; this file describes only the
state of this checkout.
