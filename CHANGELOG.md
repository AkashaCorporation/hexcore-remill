# Changelog

All notable changes to `hexcore-remill` will be documented in this file.

## [0.4.0] - 2026-04-19 — "Kernel Corpus Readiness"

### Added

- **FIX-023 — Synthetic NOP injection for x86 CET and ftrace preambles.** `DecodeInstruction` fails on `endbr64`/`endbr32` (CET landing pads, bytes `F3 0F 1E FA/FB`) and on `call __fentry__` (ftrace hook, bytes `E8 00 00 00 00` with unresolved displacement). Phase 1 now intercepts these at decode time and injects a `kCategoryNoOp` placeholder of the correct size (4 or 5 bytes) so the scan continues instead of aborting at the function preamble. Phase 2 recognizes these synthetic NOPs (`firstByte == 0xF3 || 0xE8`, size 4–5) and skips `LiftIntoBlock` for them — the TypeScript side no longer needs to pre-strip these bytes.
- **FIX-024 — XED-ILD desync recovery for exotic x86 and AArch64.** When `DecodeInstruction` fails on an instruction without a full Remill semantic model (exotic AVX-512, APX, MPX, some SSE4/AES/SHA variants on x86; ARM64 instructions without semantics), Phase 1 now falls back to Intel XED Instruction Length Decoder (`xed-ild.lib`) on x86/x86-64 — or a fixed 4-byte advance on AArch64 — to determine the instruction length, emit a `kCategoryNoOp` placeholder of that size, and advance past the offending instruction. Previously the scan would abort at the first unsupported instruction, losing the entire rest of the function (observed: kernel modules where only ~33 of ~500 instructions were lifted). Silent safety net: no performance cost when not triggered, counters and a diagnostic `[FIX-024]` stderr line only fire when exotic ISA is encountered. Thread-safe one-time XED init via `std::call_once`.
- **FIX-025 — CALL fall-through `br` wiring.** Remill lifts `call` as if the callee returns normally — execution flows through to the next PC. Phase 3's block-terminator switch previously only wired a fall-through `br` for `kCategoryNormal` and `kCategoryNoOp`; all four CALL-family categories (`DirectFunctionCall`, `IndirectFunctionCall`, `AsyncHyperCall`, `ConditionalAsyncHyperCall`) fell into `default:` and did nothing. Without an incoming edge, the return-point BB (next leader) was reachable only through Phase 4's "add ret to orphan BBs" fallback, which severed it from the caller. LLVM DCE then removed the return-point BB and everything only reachable through it. **Observed on `kbase_jit_allocate`:** 134 leaders discovered by Pathfinder, only 7 BBs survived the lift, Helix decompiled 2,137 bytes of kernel code into 13 lines of C. Fix applied in both Phase 3 (main lift) and Phase 3.5 (gap re-lift): all four CALL categories now wire fall-through `br` to the next-PC BB when that BB exists and the caller's block has no terminator.
- **Diagnostic logging for FIX-024.** `[FIX-024] Phase 1 @0x<addr>` summary line emitted when any decode failure or XED recovery occurred, showing `decoded`/`leaders`/`scanned`/`decodeFailures`/`xedRecovered`/`xedFailed`/`truncated`. `[FIX-024] XED recovery FAILED at 0x<addr>` line emitted on truly-invalid bytes with a hex dump of the first 8 bytes for diagnosis.

### Notes

- An undocumented experimental change (locally tagged "FIX-027") that removed the `firstByte` gate in Phase 2 NOP skipping and replaced `break` with `continue` on lift failure was briefly present in the working tree between Apr 12–18. It was never committed, never documented, and regressed `kbase_jit_allocate` from 2,657 → 630 lines on the `.ll` side by silently dropping instructions and leaving BBs with broken CFG connectivity. Removed by bisect on Apr 19 — this release restores the pre-experimental Phase 2 behavior. Only the documented FIX-023 / FIX-024 / FIX-025 changes ship.

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
