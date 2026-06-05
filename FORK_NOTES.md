# Fork notes (BartmanAbyss / vscode_vamiga_debugger)

This branch carries debugger-specific additions on top of `grahambates/vAmigaWeb`
(itself on top of `vAmigaWeb/vAmigaWeb`). To keep upstream merges painless, **all
fork-local logic lives in new files that don't exist upstream**, and edits to
upstream files are kept to a few one-line hooks, each tagged with a unique marker
so a single grep finds every intrusion:

```
grep -rn "vscode-vamiga-debugger host bridge" .
```

## Host bridge (UaeLib trapdoor)

A WinUAE-compatible host-call trapdoor at guest address `0xf0ff60`. The guest calls
it like a function pointer; a line-A opcode (`0xa00e`) there springs an inline Moira
software trap serviced by the host. Phase 1 = warp control via the UaeConf (`arg0=82`)
config interface; the UaeLib debug interface (`arg0=88`, overlay/graphics debugger)
is stubbed for later. Existing WinUAE debug binaries run unmodified.

**New, fork-local files (no merge surface):**
- `Core/HostBridge/HostBridge.h`
- `Core/HostBridge/HostBridge.cpp`
- `Core/HostBridge/CMakeLists.txt`

**Hooks in upstream files** (marker `// [vscode-vamiga-debugger host bridge]`):

| File | Site | Hook |
|------|------|------|
| `Core/CMakeLists.txt` | subdirectory list | `add_subdirectory(HostBridge)` |
| `Core/Components/Memory/Memory.cpp` | include block | `#include "HostBridge.h"` |
| `Core/Components/Memory/Memory.cpp` | `spypeek16<Accessor::CPU, MemSrc::NONE>` | `if (auto v = HostBridge::peek16(addr)) return *v;` — synthesizes `0xa00e`/`RTS` at the trapdoor for unmapped reads (covers the guest's presence check *and* the opcode fetch) |
| `Core/Components/CPU/CPU.cpp` | include block | `#include "HostBridge.h"` |
| `Core/Components/CPU/CPU.cpp` | `Moira::didReachSoftwareTrap` | `if (HostBridge::dispatch(*this, mem, amiga, addr)) return;` — services the trap inline (no CPU halt) before the existing `SWTRAP_REACHED` path |
| `Core/Components/CPU/CPU.cpp` | `CPU::_didReset` (hard) | `HostBridge::install(debugger.swTraps);` — registers the `0xa00e` line-A trap once |

**Re-seating after a messy merge:** the table above lists every site. The peek hook
must sit at the top of the unmapped-read `spypeek16` specialization; the dispatch
hook must run *before* `setFlag(RL::SWTRAP_REACHED)`; the install call belongs in the
`hard` reset branch after `Moira::reset()`. Everything else is self-contained in
`Core/HostBridge/`.

## CPU profiler

A per-instruction CPU profiler ported from vscode-amiga-debug / WinUAE. While
enabled it records, for each instruction in the loaded program's text range, the
reconstructed call stack (DWARF CFA unwinding of A5/A7) plus the cycle delta, into
a flat `u32` stream the host symbolicates into a call tree / flame graph. Capture is
gated by a new Moira state flag and bracketed at frame boundaries.

**New, fork-local files (no merge surface):**
- `Core/Profiler/CpuProfiler.h`
- `Core/Profiler/CpuProfiler.cpp`
- `Core/Profiler/CMakeLists.txt`

**Hooks in upstream files** (marker `// [vscode-vamiga-debugger cpu profiler]`):

| File | Site | Hook |
|------|------|------|
| `Core/CMakeLists.txt` | subdirectory list | `add_subdirectory(Profiler)` |
| `Core/Components/CPU/Moira/MoiraTypes.h` | `namespace State` | `PROFILING = (1 << 10)` — first free flag bit (8/9 are CHECK_WP/CHECK_CP) |
| `Core/Components/CPU/Moira/Moira.h` | clock accessors | `enableProfiling()/disableProfiling()` — set/clear the flag (forces the execute() slow path) |
| `Core/Components/CPU/Moira/Moira.cpp` | include block | `#include "CpuProfiler.h"` |
| `Core/Components/CPU/Moira/Moira.cpp` | `execute()` after the `LOGGING` block | `if (flags & PROFILING) CpuProfiler::beginInstr(reg.pc0, reg.a[5], reg.sp, clock);` |
| `Core/Components/CPU/Moira/Moira.cpp` | `execute()` at the `done:` label | `if (flags & PROFILING) CpuProfiler::endInstr(clock);` |
| `main.cpp` | include block | `#include "CpuProfiler.h"` |
| `main.cpp` | near `wasm_write_memory` | `wasm_profile_set_unwind/start/stop/get_data` wasm exports |
| `CMakeLists.txt` (top level) | `EXPORTED_FUNCTIONS` | the four `_wasm_profile_*` symbols |

The begin/end hooks sit in the slow path only (the PROFILING flag forces it, like
LOGGING); `beginInstr` snapshots pre-execution PC/A5/A7+clock, `endInstr` computes
the cycle delta and unwinds. Interrupt/exception paths that `goto done` skip
`beginInstr`, so `endInstr` no-ops (its pending flag is clear). All real logic is in
`Core/Profiler/`.

## Other fork infrastructure
- Build fix for recent emsdk: `'HEAPU8','HEAPF32'` added to `EXPORTED_RUNTIME_METHODS`
  in the top-level `CMakeLists.txt` (commit "fix build with latest emsdk").
- Windows build steps in `BUILD_INSTRUCTIONS.md`.
