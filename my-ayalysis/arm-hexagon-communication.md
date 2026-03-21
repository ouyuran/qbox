# ARM CPU to Hexagon DSP Communication in QBox

## Why Two QEMU Instances Are Required

QEMU is compiled per-ISA target (`aarch64-softmmu`, `hexagon-softmmu`, etc.). A single QEMU instance cannot simulate two different ISA architectures simultaneously. There is no upstream QEMU machine that combines ARM and Hexagon CPUs. Therefore, ARM+Hexagon platforms in QBox require two separate `QemuInstance` objects, connected through the SystemC layer.

## What QBox Currently Provides

QBox models Hexagon as a standalone CPU (`qemu_cpu_hexagon` inheriting `QemuCpu`) with its own peripherals:
- `hexagon_l2vic` — L2 Vectored Interrupt Controller (1024 sources, 8 outputs to CPU)
- `qemu_hexagon_qtimer` — QTimer peripheral

There is no existing platform that instantiates both ARM and Hexagon together. All current Hexagon tests run in isolation.

## How ARM-to-Hexagon Communication Can Be Built

On real Qualcomm SoCs, the ARM application processor offloads tasks to Hexagon via **FastRPC**, which uses shared DRAM and interrupts. The same mechanism can be simulated in QBox using existing infrastructure:

```
ARM QemuInstance                    Hexagon QemuInstance
┌─────────────────┐                ┌──────────────────────┐
│  Linux + driver │                │  QuRT + RPC stub     │
│                 │                │                      │
│  GIC            │◄───interrupt───│  L2VIC irq_out       │
│  irq_out        │────interrupt──►│  L2VIC irq_in        │
│  ivshmem        │◄──SystemC ────►│  ivshmem             │
│                 │    router      │                      │
└─────────────────┘                └──────────────────────┘
                    shared memory
                  (task descriptors,
                   data buffers)
```

**Required QBox components (all already exist):**
- `ivshmem_plain` — PCI shared memory device, backed by a file-mapped memory region with `share=on`
- `SystemC router` — connects the two instances' address spaces
- `hexagon_l2vic` + ARM GIC — bidirectional interrupt delivery
- A Lua platform config instantiating both `QemuInstance` objects

**Missing pieces (must be implemented):**
- A Lua platform config wiring the above components together
- Guest software: ARM-side FastRPC driver and Hexagon-side RPC stub

## Time Synchronization Implications

Mixing ARM KVM (near-native speed) with Hexagon TCG (10-20x slower) creates a fundamental timing mismatch. See [hexagon-arm-time-sync.md](hexagon-arm-time-sync.md) for a detailed analysis. The short version:

- Both instances use the host wall clock as their virtual clock source by default
- QBox's quantum-based barrier synchronization ensures causal ordering but not timing accuracy
- Hexagon's virtual time advances at wall-clock rate regardless of how many instructions it actually executed
- icount mode cannot fix this in a mixed KVM+TCG setup because KVM does not support icount

For functional simulation (booting, driver testing, protocol validation), this loosely-timed model is acceptable. For timing-sensitive workloads, the mismatch must be accounted for in the test methodology.
