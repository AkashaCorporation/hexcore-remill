# Changelog

All notable changes to `hexcore-remill` will be documented in this file.

## [0.2.0] - 2026-03-22

### Added

- **LLVM optimization passes** — new Phase 5.5 runs SROA, mem2reg, EarlyCSE, InstCombine, SimplifyCFG, DCE, ADCE, and DSE on lifted IR. Reduces IR size by ~55% (e.g. 16,700 lines → 7,500 lines) and dramatically improves downstream decompilation quality.
- **Boundary detection** — `LiftOptions` struct with configurable limits: `maxInstructions` (default: 2000), `maxBasicBlocks` (default: 500), `maxBytes` (default: 32KB). Prevents oversized IR from overwhelming decompilers.
- **CALL target recording** — discovers and reports external function call targets in `LiftResult.callTargets` for cross-function analysis.
- **Truncation metadata** — `LiftResult` now includes `truncated`, `nextAddress`, and `truncationReason` fields for chunked lifting of large functions.
- **SSA naming fix** — Phase 6 names all unnamed LLVM values (`%v0`, `%v1`, ...) to prevent SSA numbering errors when parsing generated `.ll` files.
- **CALL fall-through leaders** — function call instructions now correctly create basic block boundaries at their fall-through addresses.

### Changed

- `liftBytes()` now accepts an optional third argument (`LiftOptions`) for boundary and optimization control.
- `LiftResult` interface extended with boundary detection fields (`truncated`, `nextAddress`, `truncationReason`, `callTargets`).
- `DoLift()` C++ signature now accepts `const LiftOptions&` parameter.
- Default behavior change: lifting now applies optimization passes and boundary limits by default. Use `{ optimizeIR: false }` to get raw unoptimized IR.

## [0.1.2] - 2026-02-15

### Fixed

- **Crash prevention** — added try/catch in C++ `DoLift` and `LiftBytesWorker::Execute` to prevent process abort on native exceptions (caused by `NAPI_DISABLE_CPP_EXCEPTIONS`).
- **Semantics path resolution** — `GetModuleHandleA` now tries both `hexcore_remill.node` and `hexcore-remill.node` naming conventions to resolve the semantics directory.
- **Prebuild loader** — `index.js` now tries both underscore and hyphen naming conventions for prebuilt binaries.

## [0.1.1] - 2026-02-14

### Fixed

- **Prebuild naming mismatch** — loader now tries `hexcore_remill.node`, `hexcore-remill.node`, and `node.napi.node` conventions.
- **Semantics packaging** — added semantics `.bc` files to prebuild archive.

## [0.1.0] - 2026-02-13

### Added

- Initial release.
- N-API bindings for Remill (Trail of Bits) machine code lifter.
- Synchronous and asynchronous `liftBytes` API.
- Architecture support: x86, x86_64 (amd64), aarch64, sparc32, sparc64, with AVX/AVX512 variants.
- Static linking of 168 libraries (LLVM 18, XED, glog, gflags, Remill).
- Windows x64 prebuilt binary.
