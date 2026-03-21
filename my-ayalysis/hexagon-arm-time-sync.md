# Time Synchronization: ARM KVM + Hexagon TCG in QBox

## Background

QBox supports mixed-accelerator platforms where an ARM CPU runs under KVM (near-native speed) and a Hexagon DSP runs under TCG (software emulation, ~10-20x slower). This document explains how time synchronization works between them and its inherent limitations.

## Hexagon as a CPU

Hexagon is modeled as a `QemuCpu` in QBox because:
- QEMU upstream treats it as a full CPU target (`target/hexagon/`)
- It has its own ISA, MMU/TLB, up to 8 hardware threads, and privilege modes
- It runs with its own interrupt controller (L2VIC) and timer (QTimer)

The `qemu_cpu_hexagon` class inherits from `QemuCpu` — QBox does not distinguish between CPU/DSP/NPU at the modeling level.

## Quantum-Based Synchronization

QBox uses a **barrier synchronization** model based on quantum time slices:

1. Each QEMU instance has a **Quantum Keeper (QK)** (shared per instance in TCG_SINGLE/COROUTINE mode, per-CPU in TCG_MULTI mode).
2. Each CPU has a **deadline timer** that fires at `vclock + quantum_ns`, kicking the CPU out of its execution loop.
3. At the quantum boundary, `sync_with_kernel()` calls `m_qk->sync()`, which blocks until all instances have reached the same SystemC time.
4. The **SystemC kernel** acts as the single master time reference for all instances.

The default sync policy is `multithread-quantum`, which enforces TLM-2.0 rule h: no instance can advance past the current quantum boundary until all others have caught up.

## Virtual Clock Sources

| Accelerator | `get_virtual_clock()` source | Tied to instruction count? |
|---|---|---|
| Hexagon TCG (icount off) | Host wall clock (`cpu_get_clock()`) | No |
| Hexagon TCG (icount on) | `icount_get()` = insn count × 2^shift | Yes |
| ARM KVM | Host wall clock (fallback, no override) | No |

In QEMU, `QEMU_CLOCK_VIRTUAL` dispatches to `cpus_get_virtual_clock()`. TCG registers `icount_get` as the override only when icount mode is enabled; KVM never registers an override, so both fall back to the host wall clock by default.

## The Fundamental Timing Problem

With wall clock mode (default), both instances report the same virtual time, but their actual execution progress is vastly different:

- ARM KVM executes ~100ns of guest instructions in 100ns of real time (≈1:1)
- Hexagon TCG executes perhaps 5–10ns worth of guest instructions in the same 100ns

Both quantum keepers advance by the same wall-clock delta, so **Hexagon's virtual time runs faster than its actual execution progress**. Guest timers inside Hexagon fire at wall-clock rate, not at the rate commensurate with how many instructions were actually executed.

## Can icount Fix This?

icount mode ties Hexagon's virtual clock to instruction count (`virtual_ns = insn_count << icount_time_shift`), which is more honest about execution progress. However, it cannot fully solve the mixed-accelerator problem:

1. **KVM does not support icount.** KVM's virtual clock is always the host wall clock. The two instances would then have fundamentally different clock bases, making alignment harder, not easier.

2. **`icount_mips_shift` requires manual calibration.** Hexagon is a VLIW architecture where each packet executes multiple operations in parallel. There is no single correct value, and TCG throughput varies with host load.

3. **icount is incompatible with TCG_MULTI.** QBox enforces this:
   ```cpp
   if (p_icount && m_tcg_mode == TCG_MULTI)
       SCP_FATAL(()) << "MULTI threading can not be used with icount";
   ```
   Falling back to single-threaded TCG makes Hexagon even slower.

icount is useful for **pure-TCG platforms** where determinism matters more than speed. For ARM KVM + Hexagon TCG, the clock bases are irreconcilable.

## Practical Implications

This is a known trade-off in virtual platform (VP) design. QBox's quantum synchronization guarantees **causal ordering** (events happen in the right order) but not **timing accuracy** (each core executes the right amount of work per time unit). This is the "loosely-timed" model in TLM-2.0 terminology.

The quantum size (`quantum_ns`) controls the precision/performance trade-off:
- Smaller quantum → more sync points → better time alignment, higher overhead
- Larger quantum → fewer sync points → Hexagon can fall further behind, lower overhead

For use cases requiring cycle-accurate co-simulation between ARM and Hexagon, a different approach (e.g., instruction-stepped simulation or a dedicated co-simulation framework) would be needed.
