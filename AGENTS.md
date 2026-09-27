# Agent notes for flow-compiler

## Goal

Move the entire Flow compiler into Flow. The transpilation layer (Flow to C) is
here and self-hosting. The MLIR lowering layer is being ported from the Python
host in the main `flow` repository. Track that port in `docs/MLIR_PORT.md`.

## Bootstrap C is a fixed point

The checked-in `compiler/bootstrap/flowc_stage_a.c` must stay byte-identical to
what the compiler emits from `compiler/src/main.flow` in bundle mode. When you
edit any file under `compiler/src/`, regenerate the bootstrap C before
committing, or the fixed-point check fails.

### Regeneration

```bash
# 1. Build a temporary bootstrap binary from the CURRENT checked-in C
cc -O2 -o compiler/build/flowc_bootstrap compiler/bootstrap/flowc_stage_a.c

# 2. Emit main.flow in bundle mode
FLOWC_BUNDLE=1 FLOWC_DIR=compiler/src \
  FLOWC_IN=compiler/src/main.flow FLOWC_OUT=compiler/build/bootstrap_regen.c \
  compiler/build/flowc_bootstrap

# 3. Verify the fixed point: the new binary emits the same C
cp compiler/build/bootstrap_regen.c compiler/bootstrap/flowc_stage_a.c
cc -O2 -o compiler/build/flowc_bootstrap compiler/bootstrap/flowc_stage_a.c
FLOWC_BUNDLE=1 FLOWC_DIR=compiler/src \
  FLOWC_IN=compiler/src/main.flow FLOWC_OUT=/tmp/verify.c \
  compiler/build/flowc_bootstrap
cmp -s compiler/bootstrap/flowc_stage_a.c /tmp/verify.c \
  && echo "FIXED POINT OK" || echo "DRIFT"
```

## Compiling one file

```bash
BOOT=compiler/build/flowc_bootstrap
FLOWC_BUNDLE=1 FLOWC_DIR=. FLOWC_IN=<file>.flow FLOWC_OUT=/tmp/out.c "$BOOT"
cc -O0 -o /tmp/out /tmp/out.c -Itests/lang
```

## Language corpus

`tests/lang/*.flow` is the self-host regression target. Current baseline is 104
of 126 files compiling and running. The 22 that do not are the known gaps:
closures, effects, generics, time blocks. CI fails if the passing count drops
below the baseline. Raise the baseline as gaps close. Never lower it.

## Writing Flow

Follow the idioms in the main `flow` repository and the `flow-skills` templates:
`let` versus `let mut`, explicit types, `ptr<T>` for pointers, structs and enums
over ad hoc records. When you add a module under `compiler/src/mlir/`, match the
style of the existing modules in `compiler/src/`.

## Copy

No em dashes or en dashes. No "X, not Y" constructions. Plain, spare register.
Straight quotes only.
