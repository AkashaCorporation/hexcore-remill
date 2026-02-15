# Changelog

All notable changes to `hexcore-remill` will be documented in this file.

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
