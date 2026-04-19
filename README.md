# hexcore-remill

N-API bindings for a **HikariSystem fork of [Remill](https://github.com/lifting-bits/remill)** — lifts machine code to LLVM IR bitcode, customized for the HexCore-Helix decompiler pipeline.

Part of [HikariSystem HexCore](https://github.com/LXrdKnowkill/HikariSystem-HexCore).

## Relationship to upstream Remill

This package is **not a thin wrapper** over upstream Remill. The N-API scaffold was originally inspired by the public Remill project (Trail of Bits), but `DoLift` has diverged substantially to support the specific needs of HexCore-Helix as an end-to-end decompiler. In the upstream Remill + Anvill pipeline, a binary is lifted by Remill (~200–230k lines of very verbose IR for a moderate function) and then Anvill re-shapes that IR using an externally-supplied specification (function boundaries, calling conventions, stack layout). We don't ship Anvill.

Instead, the scaffold in this fork has been extended to do part of what Anvill would normally do at the pre-lift stage. CFG discovery, basic-block leader collection, function-boundary detection, and jump-table target resolution are handled by **Pathfinder**, a sibling extension (`hexcore-pathfinder`) that runs before Remill and feeds the discovered leaders into `LiftOptions.additionalLeaders`. Everything downstream of Remill — variable recovery, stack-frame recovery, calling-convention recovery, type inference, struct-field reconstruction — is handled by **Helix** (`HexCore-Helix`), the MLIR-based decompiler engine that consumes the IR we produce here. So `Pathfinder → this fork → Helix` together fill the role that `Anvill + Remill` fills in the Trail of Bits pipeline.

Because this is the specific configuration we target, the fork also implements desync-recovery, synthetic-NOP handling, and CFG-completeness fixes that aren't in upstream Remill. These are documented in `CHANGELOG.md` under the `FIX-0NN` tags.

## Supported Architectures

| Architecture | Variants |
|---|---|
| x86 (32-bit) | `x86`, `x86_avx`, `x86_avx512` |
| x86-64 | `amd64`, `amd64_avx`, `amd64_avx512` |
| AArch64 | `aarch64`, `aarch64_little_endian` |
| SPARC | `sparc32`, `sparc64` |

## Usage

```javascript
const { RemillLifter, ARCH } = require('hexcore-remill');

const lifter = new RemillLifter(ARCH.AMD64);

// push rbp; mov rbp, rsp; pop rbp; ret
const code = Buffer.from([0x55, 0x48, 0x89, 0xe5, 0x5d, 0xc3]);
const result = lifter.liftBytes(code, 0x401000);

if (result.success) {
  console.log(result.ir);            // LLVM IR text
  console.log(result.bytesConsumed); // 6
}

lifter.close();
```

### Async (non-blocking)

```javascript
const result = await lifter.liftBytesAsync(largeBuffer, 0x140001000);
```

### Windows ABI context

```javascript
const lifter = new RemillLifter(ARCH.AMD64, OS.WINDOWS);
```

### Pathfinder integration

```javascript
const result = lifter.liftBytes(code, 0x3A20, {
  mode: 'ElfRelocatable',
  additionalLeaders: pathfinderLeaders,  // BB entry points from pre-lift CFG analysis
  knownFunctionEnds: pathfinderEnds,     // function boundaries (advisory)
  maxBytes: 65536,
  maxInstructions: 5000,
});
```

`additionalLeaders` is the main integration point with Pathfinder — every address in this list is inserted into the Phase 1.5 leaders set before the lift loop runs, so Pathfinder-discovered BBs that wouldn't be found by purely-sequential decoding (jump-table targets, `.pdata` function entries, ELF symtab addresses) get their own basic block in the output IR.

## API

### `new RemillLifter(arch, os?)`

Create a lifter for the given architecture. Loads the Remill semantics module.

- `arch` — Architecture name (use `ARCH` constants)
- `os` — OS name for ABI context (optional, defaults to `'linux'`)

### `lifter.liftBytes(code, address, options?) → LiftResult`

Synchronous lift. Decodes and lifts instructions from the buffer, starting at `address`.

Pass an optional third `options` object to control lift limits and IR shape. Named semantic helper calls are preserved by default for downstream decompiler compatibility. Set `inlineSemantics: true` only when you explicitly want the semantic helper bodies inlined into the lifted function.

### `lifter.liftBytesAsync(code, address, options?) → Promise<LiftResult>`

Async lift in a worker thread. Use for large buffers (>64KB).

### `LiftOptions`

```typescript
{
  maxInstructions?: number;         // default 2000
  maxBasicBlocks?: number;          // default 500
  maxBytes?: number;                // default 32768 (32KB)
  splitAtCalls?: boolean;           // default true
  optimizeIR?: boolean;             // default true (SROA, mem2reg, InstCombine, SimplifyCFG, DCE, ADCE, DSE)
  inlineSemantics?: boolean;        // default false (preserve named semantic helper calls)
  mode?: 'Generic' | 'PE64' | 'ElfRelocatable';  // format-specific heuristics
  additionalLeaders?: number[];     // extra BB entry points from external analysis (Pathfinder)
  knownFunctionEnds?: number[];     // function end addresses (advisory, used for tail-call detection)
}
```

### `LiftResult`

```typescript
{
  success: boolean;
  ir: string;              // LLVM IR text
  error: string;           // Error message if !success
  address: number;         // Start address
  bytesConsumed: number;   // Bytes consumed from input
  truncated: boolean;      // true if a limit was hit before all bytes consumed
  nextAddress: number;     // where to continue lifting (valid if truncated)
  callTargets: number[];   // discovered external CALL targets
  truncationReason: string; // "max_instructions" | "max_blocks" | "max_bytes"
  implicitParams: string[]; // registers read before written (function params)
}
```

### `RemillLifter.getSupportedArchs() → string[]`

Returns list of supported architecture names.

## Pipeline Position

```
┌──────────────┐   ┌──────────────────┐   ┌──────────────┐   ┌──────────────┐
│ hexcore-     │   │                  │   │              │   │              │
│ disassembler │→→→│   Pathfinder     │→→→│  this fork   │→→→│    Helix     │
│ (ELF/PE)     │   │   (CFG leaders,  │   │  (Remill     │   │  (decompile  │
│              │   │    boundaries)   │   │   lifting)   │   │   to C)      │
└──────────────┘   └──────────────────┘   └──────────────┘   └──────────────┘
                             │                    │
                             └────── spec ────────┘
                          (analogous to Anvill's JSON spec in
                           the Trail of Bits reference pipeline)
```

## Building from Source

```bash
# Prerequisites: LLVM 18 static libs, Node 18+, Windows 10 SDK + MSVC (Windows),
# clang-cl / MSVC (whichever is in your node-gyp toolchain)

# Build Remill + XED deps first (see deps/README.md or unpack remill-deps-win32-x64.zip)
npx node-gyp rebuild
npm test
```

A prebuilt binary is shipped at `prebuilds/win32-x64/hexcore_remill.node`. The loader in `index.js` prefers a local `build/Release` artifact when present (useful during development), falling back to the prebuilt otherwise.

## Dependencies

- [Remill](https://github.com/lifting-bits/remill) — static library, patched fork (see `CHANGELOG.md` for the `FIX-0NN` series)
- [LLVM 18](https://llvm.org/) — static libraries (Core, Support, BitReader, BitWriter, IRReader, etc.)
- [Intel XED](https://github.com/intelxed/xed) — x86 instruction decoder (used by Remill and by FIX-024 desync recovery)

**Important:** Must use the same LLVM version as `hexcore-llvm-mc` (currently LLVM 18) to avoid symbol conflicts when both are loaded in the same process.

## License

MIT — Copyright (c) HikariSystem.

Upstream Remill (Trail of Bits) is Apache-2.0. This fork preserves the upstream copyright notices in `deps/remill/`.
