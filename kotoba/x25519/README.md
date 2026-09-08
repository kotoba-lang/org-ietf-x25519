# The Kotoba guest

`field.kotoba` and `core.kotoba` are X25519 as a pure Kotoba guest, with
`src/x25519/` as the oracle. They are checked in a way worth stating, because
the revision before this one was not: **each has a `main` that runs known
answers and returns the NUMBER of bytes that disagree.**

A count rather than a boolean, so one regression is distinguishable from a
build that answers nothing right.

## How to run them

```bash
AMU=<path to kotoba-lang/amu>/bin/amu

# the field: 0 on wasm32 and as a native aarch64 binary
$AMU compile kotoba/x25519/field.kotoba --jvm-free --target wasm32 \
     --fuel 60000000 --output field.wasm
node --stack-size=60000 -e 'import("'$PWD'/field-run.mjs")'   # main() = 0

# the ladder: RFC 7748 section 5.2 test vector 1
$AMU compile kotoba/x25519/core.kotoba --source-path kotoba --jvm-free \
     --target wasm32 --fuel 900000000 --output core.wasm
```

Compiling `core.kotoba` takes about four minutes, nearly all of it the
compile-time oracle actually running a 255-step scalar multiplication. What it
seals is `main`'s answer, and it is 0.

## What runs where, measured 2026-09-08

| | KIR oracle | wasm32 host | native aarch64 |
|---|---|---|---|
| `field.kotoba` | 0 | 0 | **0** |
| `core.kotoba` | 0 | vector arena | vector arena |

The ladder's semantics are established -- the oracle executes the whole
program -- but neither host will *run* it yet, and the reason is one bound
rather than anything about the code. Every write to an immutable vector mints
a new one; amu's `tools/kexe_loader.c` caps the vector arena
(`KEXE_VECTOR_CAPACITY` 4096, `KEXE_VECTOR_ITEM_CAPACITY` 65536) and the
browser host has the same ceiling. One ladder step is on the order of a
thousand vectors and there are 255 of them. That file says raising the bound
"is a separate decision with its own fail-closed argument, and it has not been
made"; this is a use for it, not a decision to make here.

The field module is under that ceiling and does run: `extract-native` plus
`kexe_loader` answers 0, and 1 with a single expected byte perturbed.

## Two things a reader will otherwise assume

**Only i64-typed entries can be exported natively.** Every one of these
functions compiles for aarch64 -- measured, definition by definition -- but a
module exporting `pack`, which returns a `:vector-i64`, is refused at the
target gate. The native variant used for the measurement above exports `main`
alone.

**`:document` is not an option here.** The native backend admits no typed
document CONSTRUCTOR: `(document-i64 42)` on its own is refused for
aarch64-macos. That is why these two files are `:vector-i64`.
