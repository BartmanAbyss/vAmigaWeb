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
| `Core/Components/CPU/Moira/Moira.cpp` | `execute()` after the `LOGGING` block | `if (flags & PROFILING) CpuProfiler::beginInstr(reg.pc0, reg.a[5], reg.sp, reg.usp, reg.sr.s, clock);` |
| `Core/Components/CPU/Moira/Moira.cpp` | `execute()` at the `done:` label | `if (flags & PROFILING) CpuProfiler::endInstr(clock);` |
| `Core/Components/CPU/Moira/MoiraExec_cpp.h` | `execJsr` (both branches), after `push(reg.pc)` | `if (flags & State::PROFILING) CpuProfiler::BranchStack::push(reg.sr.s, reg.pc, reg.sp);` |
| `Core/Components/CPU/Moira/MoiraExec_cpp.h` | `execBsr` (both branches), after `push(retpc)` | `if (flags & State::PROFILING) CpuProfiler::BranchStack::push(reg.sr.s, retpc, reg.sp);` |
| `Core/Components/CPU/Moira/MoiraExec_cpp.h` | `execRts`, after `setPC(newpc)` (cycle row `16,16,10`) | `if (flags & State::PROFILING) CpuProfiler::BranchStack::popRts(reg.sr.s, newpc);` |
| `Core/Components/CPU/Moira/MoiraExec_cpp.h` | `execRte`, after `setPC(newpc)` (cycle row `20,24,20`) | `if (flags & State::PROFILING) CpuProfiler::BranchStack::popRte(newpc);` |
| `Core/Components/CPU/Moira/MoiraExceptions_cpp.h` | `execException<C>` after `setSupervisorMode(true)` | `if (flags & State::PROFILING) CpuProfiler::BranchStack::enterException(reg.pc, reg.sp);` |
| `Core/Components/CPU/Moira/MoiraExceptions_cpp.h` | `execInterrupt<C>` before `jumpToVector` | `if (flags & State::PROFILING) CpuProfiler::BranchStack::enterException(reg.pc, reg.sp);` |
| `main.cpp` | include block | `#include "CpuProfiler.h"` |
| `main.cpp` | near `wasm_write_memory` | `wasm_profile_set_unwind/start/stop/get_data` wasm exports |
| `CMakeLists.txt` (top level) | `EXPORTED_FUNCTIONS` | the four `_wasm_profile_*` symbols |

The begin/end hooks sit in the slow path only (the PROFILING flag forces it, like
LOGGING); `beginInstr` snapshots pre-execution PC/A5/A7 + S-bit + clock, `endInstr`
computes the cycle delta and reconstructs the stack. Interrupt/exception paths that
`goto done` skip `beginInstr`, so `endInstr` no-ops (its pending flag is clear). All
real logic is in `Core/Profiler/`.

**Branch-stack fallback (no-DWARF / assembly).** When the host uploads an *empty*
unwind table (a hunk program with no `.debug_frame`), `CpuProfiler::start()` selects
runtime branch-stack reconstruction instead of DWARF: the `BranchStack::push/popRts/
popRte/enterException` hooks above maintain a shadow call stack, ported 1:1 from
WinUAE's `debugmem.cpp` (`branch_stack_push` / `_pop_rts` / `_pop_rte`). Two stacks
keyed on the S-bit (USP vs SSP), pop matched by return PC; `popRte` always unwinds the
supervisor stack; a user push/pop resets the supervisor count (WinUAE cleanup); the
exception/interrupt entry hooks bridge the handler to the interrupted code (so IRQ
frames appear). Deviations from WinUAE: no per-frame `regs[16]` snapshot (we only emit
PCs), and overflow drops the oldest frame rather than resetting the whole stack.
`CpuProfiler::seedFromStack` seeds the stack at capture start by scanning for return
addresses — **this heuristic mirrors `src/stackManager.ts` `guessStack`; keep the two in
sync.** The emitted stream format is identical to the DWARF path, so the host/webview is
unaware which method ran. No new exports or hooks beyond the rows above.

## DMA profiler

A per-DMA-cycle bus profiler, sibling of the CPU profiler, captured in the **same
frame** (it rides `wasm_profile_start`). It records one 8-byte cell per dma-cycle —
`{ owner, flags, data, addr }` — into a frame-wide "enriched grid", plus a chip/slow-RAM
snapshot at capture start. The host uses the grid to draw a DMA channel line + per-channel
totals, and (future) to reconstruct memory by replaying the grid's WRITE cells over the
snapshot. The grid is built by mirroring Agnus's per-line `busOwner/busAddr/busData` at
EOL; the two things those arrays lack — read-vs-write and byte-vs-word — are added as a
`flags` byte (`WRITE|BYTE|CODE` + 2-bit Copper sub-state) stamped at the write sites.

**New, fork-local files (no merge surface):**
- `Core/Profiler/DmaProfiler.h`
- `Core/Profiler/DmaProfiler.cpp`
- (built via the existing `Core/Profiler/CMakeLists.txt`)

**Hooks in upstream files** (marker `// [vscode-vamiga-debugger dma profiler]`). All
per-cycle hooks are gated inline by `DmaProfiler::enabled()` so normal emulation pays
only a branch:

| File | Site | Hook |
|------|------|------|
| `Core/Profiler/CMakeLists.txt` | sources | `DmaProfiler.cpp` |
| `Core/Components/CPU/Moira/Moira.h` | near `enableProfiling` | `fcIsProgram()` — CPU Code/Data from the function code `fcl` |
| `Core/Components/Agnus/Copper/Copper.h` | public accessors | `dmaSubState()` — current copper command type (MOVE/WAIT/SKIP) |
| `Core/Components/Agnus/Agnus.cpp` | include block | `#include "DmaProfiler.h"` |
| `Core/Components/Agnus/Agnus.cpp` | `eolHandler` after `dmaDebugger.eolHandler()` | `recordLine(pos.v, busOwner, busAddr, busData)` — fold the line into the grid before the busOwner table is cleared |
| `Core/Components/Agnus/Agnus.cpp` | `executeUntilBusIsFree` after `busOwner = CPU` | `markCpu(pos.h, cpu.fcIsProgram())` |
| `Core/Components/Agnus/AgnusDma.cpp` | include block | `#include "DmaProfiler.h"` |
| `Core/Components/Agnus/AgnusDma.cpp` | `doCopperDmaRead` | `markCopper(pos.h, copper.dmaSubState())` |
| `Core/Components/Agnus/AgnusDma.cpp` | `doDiskDmaWrite` / `doBlitterDmaWrite` | `markWrite(pos.h, false)` |
| `Core/Components/Agnus/AgnusDma.cpp` | `doCopperDmaWrite` | `markWrite(pos.h,false)` + `markCopper(pos.h, COP_SUB_MOVE)` |
| `Core/Components/Memory/Memory.cpp` | include block | `#include "DmaProfiler.h"` |
| `Core/Components/Memory/Memory.cpp` | `poke8/16 <CPU,CHIP>`, `<CPU,SLOW>`, `poke16<CPU,CUSTOM>` after the busAddr/busData stamp | `markWrite(pos.h, isByte)` |
| `main.cpp` | include block | `#include "DmaProfiler.h"` |
| `main.cpp` | `wasm_profile_start` alongside `CpuProfiler::start/stop` | `DmaProfiler::setMemory/start/stop` (same frame) |
| `main.cpp` | after `wasm_profile_get_data` | `wasm_dma_get_data` / `wasm_dma_get_snapshot` exports |
| `CMakeLists.txt` (top level) | `EXPORTED_FUNCTIONS` | `_wasm_dma_get_data`, `_wasm_dma_get_snapshot` |

**Scope / known gaps (this phase):** Blitter is a single color (no per-channel/Fill/Line —
that needs `SlowBlitter` state, deferred to the blitter-visualizer phase). PAL only. The
custom-register baseline snapshot is deferred (write-only regs aren't exposed via spypeek);
copper color-register writes (`0x180..0x1BE`) bypass `doCopperDmaWrite` in vAmiga so they
aren't in the grid; FAST-RAM writes bypass the chip bus. Reconstruction (chip+slow RAM +
custom registers) is data-only/unwired this phase.

## Other fork infrastructure
- Build fix for recent emsdk: `'HEAPU8','HEAPF32'` added to `EXPORTED_RUNTIME_METHODS`
  in the top-level `CMakeLists.txt` (commit "fix build with latest emsdk").
- Windows build steps in `BUILD_INSTRUCTIONS.md`.
