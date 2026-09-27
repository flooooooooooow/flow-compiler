# flow-compiler

The Flow language compiler, written in Flow.

This repository holds the self-hosted `flowc`. The compiler that compiles Flow
is itself a Flow program. The aim is that the whole pipeline lives here in Flow:
the front end (lexer, parser, type checker), the C transpilation layer, and the
MLIR lowering layer. The Python host in the main [`flow`](https://github.com/flooooooooooow/flow)
repository stays as the reference until each layer is ported and verified here.

## Status

- Transpilation layer (Flow to portable C): present and self-hosting. The
  bootstrap compiles itself to a three-generation byte-identical fixed point.
- Language corpus: 103 of 126 `tests/lang` files compile and run through the
  self-hosted compiler on the Linux CI runner (104 on macOS; the one difference,
  `test_geometry_diagram`, is platform-dependent). The rest are the known gaps
  (closures, effects, generics, time blocks) tracked for the port.
- MLIR lowering layer: in Flow and growing. `compiler/src/mlir_gen.flow` emits
  the `func`, `arith`, `scf`, and `memref` dialects under `FLOWC_BACKEND=mlir`.
  Faithful coverage: integer arithmetic and comparisons, local bindings and
  assignments, unary minus, integer function calls (`func.call`), pure-float
  functions (`arith.addf`/etc), and control flow (`if`/`while`/`for` via memref
  slots and `scf`). Anything outside the covered subset emits a valid typed
  stub. `mlir-opt` verifies the output. The full plan is in
  [`docs/MLIR_PORT.md`](docs/MLIR_PORT.md).

## Build

The checked-in `compiler/bootstrap/flowc_stage_a.c` is the seed. Build it with
any C11 compiler, then use it to compile Flow.

```bash
mkdir -p compiler/build
cc -O2 -o compiler/build/flowc_bootstrap compiler/bootstrap/flowc_stage_a.c
```

Compile one Flow file to C and run it:

```bash
BOOT=compiler/build/flowc_bootstrap
FLOWC_BUNDLE=1 FLOWC_DIR=. FLOWC_IN=tests/lang/test_arithmetic.flow \
  FLOWC_OUT=/tmp/out.c "$BOOT"
cc -O0 -o /tmp/out /tmp/out.c -Itests/lang
/tmp/out
```

Run the whole language corpus:

```bash
BOOT=compiler/build/flowc_bootstrap
for f in $(find tests/lang -name '*.flow' | sort); do
  FLOWC_BUNDLE=1 FLOWC_DIR=. FLOWC_IN="$f" FLOWC_OUT=/tmp/out.c "$BOOT" \
    && cc -O0 -o /tmp/out /tmp/out.c -Itests/lang \
    && echo "ok   $f" || echo "FAIL $f"
done
```

## Layout

| Path | Contents |
|------|----------|
| `compiler/src/` | The compiler, in Flow: lexer, parser, type checker, C backend |
| `compiler/src/mlir_gen.flow` | The MLIR lowering backend (func/arith), in Flow |
| `compiler/bootstrap/` | The seed C (`flowc_stage_a.c`) and its built binary |
| `compiler/scripts/` | Bootstrap regeneration and self-host verification |
| `compiler/host/` | The C driver that hosts the Stage-A compiler |
| `tests/lang/` | The language corpus, the self-host regression target |
| `docs/MLIR_PORT.md` | The Python-to-Flow port plan for the MLIR layer |

## Bootstrap regeneration

Editing anything under `compiler/src/` means the checked-in bootstrap C must be
regenerated so it stays a fixed point. See [`AGENTS.md`](AGENTS.md).

## Continuous integration

CI runs on GitHub Actions because this repository is public and the free runner
keeps the build off local machines. It builds the bootstrap and runs the corpus
against a baseline that fails on regression.

## License

Proprietary. All rights reserved. See [`LICENSE`](LICENSE). The source is
published for visibility and CI. For licensing inquiries, contact
abhishek.shivakumar@gmail.com.
