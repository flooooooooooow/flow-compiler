# MLIR layer port plan

The Flow compiler emits C today, and the MLIR lowering still lives in Python
under `src/flow/`. This document tracks moving that layer into Flow so the whole
compiler is written in Flow. It lists every Python source, its target Flow
module, a rough size, and a checkbox, and it groups the work into phases that
can land one at a time.

The port follows the style already set in `compiler/src/`: index-based arenas,
integer tag constants, explicit types, `ptr<T>` for low-level work. It lands as
a flat backend module, `compiler/src/mlir_gen.flow`, the same shape as
`jsgen.flow` and `cgen.flow`: a fixed-buffer writer and an AST-arena walk that
emits text directly. That is the pattern the self-hosted compiler already
proves it can compile, so the port builds on it rather than on a new
subdirectory layout.

## Status

`compiler/src/mlir_gen.flow` is wired into `main.flow` under
`FLOWC_BACKEND=mlir` and ships in the checked-in bootstrap (fixed point
verified). It emits one `func.func` per Flow function with typed block arguments
and a typed return.

Faithful lowering covers straight-line integer and bool functions: parameters,
`i32` local `let` bindings, assignments (rebound in SSA form since there are no
control-flow merges yet), integer arithmetic (`arith.addi`/`subi`/`muli`/
`divsi`/`remsi`), unary minus, integer comparisons (`arith.cmpi`), and a
trailing return. Every other function emits a valid typed zero-return stub, so
the whole module is always MLIR the tools accept. `mlir-opt` parses, verifies,
and canonicalizes the output.

Pure-float functions (all parameters and the return one of `f64`/`f32`) whose
body is a single return also lower faithfully, to `arith.addf`/`subf`/`mulf`/
`divf`/`negf` with float constants. Mixed int and float signatures still stub.

Integer function calls lower to `func.call` when the callee is a defined,
non-generic i32 function (all i32 parameters, i32 return), so integer helpers
compose across the module.

Control flow lowers through memory. When an integer function contains `if`,
`while`, or `for`, each parameter and local becomes a `memref<i32>` stack slot;
reads are `memref.load`, writes are `memref.store`, and the control flow is
`scf.if` and `scf.while` with no loop-carried SSA values because the mutable
state lives in memory. `for` lowers to a `scf.while` over an induction slot with
the bound evaluated once. Nested control flow works. The one restriction is no
early return: `return` is allowed only as the final statement. `mlir-opt`
verifies the output, and its `mem2reg`/`sccp` passes can raise the slots back to
SSA when wanted.

Still to do: multi-statement floats, float and mixed calls, early return, then
the canonicalize, optimizer, backend, and JIT phases below.

## Source-to-target map

Line counts are of the current Python source. They give a rough guide to
effort. The Flow port will land at a different size.

| Python source | Lines | Target Flow module | Phase | Done |
|---|---|---|---|---|
| `src/flow/mlir_generator.py` (type mapping, struct helpers) | 7442 | `compiler/src/mlir/mlir_types.flow` | 1 | [ ] |
| `src/flow/mlir_generator.py` (per-op emission, SSA helpers) | (same file) | `compiler/src/mlir/mlir_ops.flow` | 2 | [ ] |
| `src/flow/mlir_generator.py` (`MLIRGenerator`, `flow_to_mlir`) | (same file) | `compiler/src/mlir/mlir_emit.flow` | 2 | [ ] |
| `src/flow/mlir_canonicalize.py` | 322 | `compiler/src/mlir/mlir_canonicalize.flow` | 3 | [ ] |
| `src/flow/mlir_optimizer.py` | 376 | `compiler/src/mlir/mlir_optimizer.flow` | 3 | [ ] |
| `src/flow/denotational_mlir.py` | 813 | `compiler/src/mlir/denotational.flow` | 3 | [ ] |
| `src/flow/metal_codegen.py` | 458 | `compiler/src/mlir/metal_codegen.flow` | 4 | [ ] |
| `src/flow/wgsl_codegen.py` | 511 | `compiler/src/mlir/wgsl_codegen.flow` | 4 | [ ] |
| `src/flow/mlir_gpu_codegen.py` | 606 | `compiler/src/mlir/gpu_codegen.flow` | 4 | [ ] |
| `src/flow/mlir_spirv.py` | 198 | `compiler/src/mlir/spirv.flow` | 4 | [ ] |
| `src/flow/gpu_integration.py` | 491 | `compiler/src/mlir/gpu_integration.flow` | 4 | [ ] |
| `src/flow/gpu_runtime.py` | 417 | `compiler/src/mlir/gpu_runtime.flow` | 4 | [ ] |
| `src/flow/metal_runtime.py` | 367 | `compiler/src/mlir/metal_runtime.flow` | 4 | [ ] |
| `src/flow/wasm_compiler.py` | 212 | `compiler/src/mlir/wasm_compiler.flow` | 4 | [ ] |
| `src/flow/mlir_jit.py` | 522 | `compiler/src/mlir/mlir_jit.flow` | 5 | [ ] |

The main repo already carries hand-ported `metal_codegen.flow` and
`wgsl_codegen.flow` in its own `compiler/src/`. Treat those as a reference for
the Phase 4 backends rather than starting from the Python each time.

## Phases

### Phase 1: core IR and types

Target: `compiler/src/mlir/mlir_types.flow`.

Port `MLIRGenerator.flow_type_to_mlir` (around line 6695 of
`mlir_generator.py`) and the struct and tensor helpers near the top of that
file. This is the foundation every later phase reads.

The Python emitter carries MLIR types as strings. Model them as tagged records
in an arena instead:

- Integers, `i1` for `bool`, and the Flow `uN` to `iN` fold (MLIR has no
  unsigned integers; zero-extension uses `arith.extui`).
- Floats `f32` and `f64`, and `void` rendered as `()`.
- `!llvm.ptr` for `string`, `ptr<T>`, and struct or pointer array elements.
- `memref<SIZExELEM>` for arrays, with a dynamic `?` extent.
- `vector<SIZExELEM>` for the `vec*` types.
- `!flow.struct<Name>` for structs, and the raw `!llvm.struct<...>` descriptor.
- The function type `(params...) -> ret`.

Entry points: `flowc_flow_type_to_mlir`, `flowc_mlir_type_render`,
`flowc_mlir_type_is_aggregate`, plus the per-kind constructors.

Verify by rendering each type kind back to the exact string the Python emitter
produces for the same Flow type.

### Phase 2: op emission

Targets: `compiler/src/mlir/mlir_ops.flow` and
`compiler/src/mlir/mlir_emit.flow`.

`mlir_ops.flow` holds one op record per emitted line, with a result SSA number,
a result type id, and up to three operand SSA numbers. That covers the constant,
unary, binary, load, store, branch, and call forms. The op set the emitter
reaches for:

- `arith`: `constant`, `addi`, `subi`, `muli`, `addf`, `subf`, `mulf`, `divf`,
  `cmpi`, `cmpf`, `extui`, `extsi`, `extf`, `index_cast`.
- `llvm`: `mlir.constant`, `mlir.undef`, `mlir.zero`, `alloca`, `load`, `store`,
  `getelementptr`, `call`.
- `memref`: `alloc`, `load`, `store`.
- `func` and `cf`: `func.call`, `func.return`, `cf.br`, `cf.cond_br`.

`mlir_emit.flow` ports the `MLIRGenerator` walk and the `flow_to_mlir`
top-level. It threads an `MlirEmitter` holding the SSA counter
(`function_counter`), the indent depth, the target word size, and the two
arenas. Statement emitters append to the current block; expression emitters
return an SSA number and append their ops. Entry points mirror the Python
public methods: `generate_module`, `generate_function`, `generate_block`,
`generate_statement`, `generate_var_decl`, `generate_return`, `generate_if`,
`generate_while`, `generate_for`, `generate_expression`, `generate_literal`,
`generate_variable`, `generate_binary_operation`, `generate_function_call`, and
`flowc_flow_to_mlir`.

Two target-shape details to carry over: aggregates spill to `llvm.alloca` slots
to dodge arm64 return-slot aliasing, and that spill is skipped on 32-bit targets
because it conflicts with ASYNCIFY stack rewind (flow#467).

Verify against the Python emitter output for a spread of the `tests/lang`
programs: same ops, same SSA numbering, same types.

### Phase 3: canonicalize and optimize

Targets: `mlir_canonicalize.flow`, `mlir_optimizer.flow`, `denotational.flow`.

- `mlir_canonicalize.py` rewrites counted `while` loops into a canonical form
  and finds trivial accessors. Port `canonicalize_counted_loops` and
  `find_trivial_accessors`.
- `mlir_optimizer.py` wraps `mlir-opt`. Port `MLIROptimizer` with
  `build_pass_pipeline`, `optimize`, `analyze_vectorization`, and
  `get_optimization_report`. The pipeline construction is pure string work; the
  tool invocation goes through the existing process helpers.
- `denotational_mlir.py` emits the `flow.*` dialect that keeps the vector-field
  structure of evolution blocks so the fusion passes can act on them. Port
  `emit_denotational_mlir`, `emit_ensemble_step`, `emit_ensemble_step_func`, and
  `emit_ensemble_run_module`. This is the lane `flowc_flow_to_mlir` prepends
  when the denotational flag is set.

### Phase 4: backends

Targets: the GPU and wasm modules.

Each backend reads the emitted MLIR or the declaration list and produces a
target artifact:

- `metal_codegen.flow` and `wgsl_codegen.flow`: shader source per GPU function.
  `extract_gpu_functions` selects the functions; the generator walks each one.
- `gpu_codegen.flow` and `spirv.flow`: the MLIR gpu module path and the SPIR-V
  translation through `mlir-opt` and `mlir-translate`.
- `gpu_integration.flow`, `gpu_runtime.flow`, `metal_runtime.flow`: backend
  selection, availability checks, and the runtime launch glue.
- `wasm_compiler.flow`: `flow_to_wasm` and `llvm_to_wasm`, which run clang and
  validate the exported symbols.

Verify each backend against a golden shader or module for a known GPU function,
and check the wasm exports match the declared set.

### Phase 5: JIT

Target: `mlir_jit.flow`.

Port `MLIRJIT` and `FlowJITRuntime`: `compile_mlir_to_llvm`,
`compile_llvm_to_executable`, `compile_llvm_to_native`, `execute_function`, and
`jit_compile_and_run`, plus the runtime link-arg and exit-code helpers. This
sits on top of everything above, so it lands last.

Verify by compiling and running a small MLIR module and checking the exit code
and any returned value.

## Verification for the whole port

Climb only as far as the risk requires, the same ladder the compiler already
uses:

1. Each Flow module parses and emits C.
2. The C compiles.
3. The ported stage produces the same MLIR text as the Python emitter for the
   `tests/lang` corpus.
4. The MLIR compiles and runs through the JIT with the expected result.

Match against the Python output byte for byte where the two are meant to agree,
and record any place they diverge on purpose.
