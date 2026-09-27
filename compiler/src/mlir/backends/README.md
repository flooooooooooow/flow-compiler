# MLIR backends

Flow port skeletons for the backend lowering targets. Today these targets live
in Python under `src/flow/` in the main Flow repository. The goal is to move
them into Flow so the whole compiler is self-hosted. Each file here fixes the
struct and signature shapes; the bodies are stubs marked `TODO(port)` that name
the Python source they come from.

## Backends

- `spirv.flow`. SPIR-V lowering for @gpu kernels. Ported from
  `mlir_gpu_codegen.py` (the gpu.module emitter) and `mlir_spirv.py` (the
  mlir-opt SPIR-V pipeline and mlir-translate serialization). Status: skeleton.

- `metal.flow`. Metal Shading Language codegen for @gpu kernels. Ported from
  `metal_codegen.py` (kernel emit, Objective-C host emit, buffer bindings).
  Status: skeleton.

- `wgsl.flow`. WebGPU Shading Language codegen for @gpu kernels. Ported from
  `wgsl_codegen.py` (kernel emit, storage vs uniform bindings, the padded
  Params block, the binding layout). Status: skeleton.

- `wasm.flow`. WebAssembly lowering for Flow source. Ported from
  `wasm_compiler.py` (Flow to LLVM IR to wasm32, export validation, link
  options). Status: skeleton.

## Notes

The Metal, WGSL, and SPIR-V codegen stages are pure text emission from the AST
and belong entirely in Flow. The SPIR-V serialization and the WASM link step
drive external LLVM/MLIR tools, so those stages stay host concerns. The
skeletons carry the Flow-side contract around them.
