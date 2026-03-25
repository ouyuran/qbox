# QNN on ARM+Hexagon: QBox Simulation Guide

## How QBox Supports Hexagon NPU

QBox models Hexagon across three layers:

1. **libqemu-cxx** — `qemu::CpuHexagon` wraps QEMU's Hexagon C API. Supported DSP revisions: v66, v68, v69, v73, v79, v81.
2. **libqbox** — SystemC TLM-2.0 components:
   - `qemu_cpu_hexagon` — CPU model with HVX, VTCM, multi-thread support
   - `hexagon_globalreg` — boot configuration device (config table, qtimer base, boot entry vector)
   - `hexagon_l2vic` — L2 Vectored Interrupt Controller (1024 sources, 8 CPU outputs)
   - `qemu_hexagon_qtimer` — Qualcomm QTimer peripheral
3. **QEMU** — `hexagon-softmmu` target, TCG instruction translation

ARM and Hexagon require **two separate `QemuInstance` objects** because QEMU is compiled per-ISA. Both instances share a single SystemC router and address space.

## QNN Runtime Flow

QNN (Qualcomm Neural Network SDK) does not use a standalone DSP firmware binary. Instead:

```
ARM: QNN Runtime (libQnnHtp.so)
  → opens /dev/fastrpc-cdsp
  → writes QuRT + HTP skeleton (libQnnHtpV68Skel.so) into Hexagon memory via fastrpc driver
  → writes hexagon_globalreg boot-evb register → triggers Hexagon CPU start
  → sends RPC calls (operator descriptors + tensor data) via shared memory
  → Hexagon: QuRT receives → executes HVX vector compute
  → results written back to shared memory
  → ARM reads results
```

There is no pre-built DSP bin to load at simulation start. The fastrpc driver (`drivers/misc/fastrpc.c`) loads `cdsp.mbn` (QuRT + skeleton) from ARM rootfs (`/lib/firmware/`) into Hexagon memory at runtime.

## Memory Address Layout

Hexagon memory regions are **hardware-defined** by the SoC's memory controller access permissions (XPU/SMMU). The DTB `reserved-memory` node describes these hardware facts — it does not define them. Example from a Qualcomm SoC DTS:

```dts
reserved-memory {
    cdsp_mem: memory@98900000 {
        reg = <0x0 0x98900000 0x0 0x1e00000>;
        no-map;
    };
};
fastrpc { memory-region = <&cdsp_mem>; };
```

Use the target platform's DTS (e.g., `arch/arm64/boot/dts/qcom/sm8550.dtsi`) for correct addresses. Do not use placeholder values in production configs.

## Simulating the fastrpc Load in QBox

No custom loader is needed. The flow works with existing QBox components:

1. ARM Linux boots with `fastrpc.ko` driver
2. fastrpc driver writes QuRT+skel into the Hexagon DDR region (via SystemC router → `hex_ddr` memory)
3. fastrpc driver writes `hexagon_globalreg.boot-evb` → QBox propagates this to QEMU as `boot-evb` property → Hexagon CPU starts executing from that address
4. QuRT initializes, FastRPC server starts, signals readiness to ARM via shared memory + interrupt

The `hex_cpu_0` must be configured with `start_powered_off = true` so Hexagon waits for ARM to trigger it.

## Minimal Lua Platform Sketch

```lua
-- Two QemuInstances sharing one router
qemu_inst_arm = {
    moduletype = "QemuInstance",
    args = { "&platform.qemu_inst_mgr", "AARCH64" },
    accel = "tcg", sync_policy = "multithread-unconstrained",
},
qemu_inst_hex = {
    moduletype = "QemuInstance",
    args = { "&platform.qemu_inst_mgr", "HEXAGON" },
    accel = "tcg", sync_policy = "multithread-unconstrained",
},

-- Hexagon DDR region (address from platform DTS, not a placeholder)
hex_ddr = {
    moduletype = "gs_memory",
    target_socket = { address = 0x98900000, size = 0x1e00000,
                      bind = "&router.initiator_socket" },
},

hex_globalreg = {
    moduletype = "hexagon_globalreg",
    args = { "&platform.qemu_inst_hex" },
    dsp_arch = "v68",
    config_table_addr  = <from_platform_dts>,
    qtimer_base_addr   = <from_platform_dts>,
    hexagon_start_addr = <from_platform_dts>,
},

hex_cpu_0 = {
    moduletype = "qemu_cpu_hexagon",
    args = { "&platform.qemu_inst_hex", "&platform.hex_globalreg" },
    mem = { bind = "&router.target_socket" },
    dsp_arch = "v68",
    start_powered_off = true,   -- ARM fastrpc driver triggers start
    hvx_contexts = 4,
    vtcm_base_addr = <from_platform_dts>,
    vtcm_size_kb = 256,
},
```

## Why Not Two Standalone QEMU Processes

Running two QEMU processes without QBox requires solving:

| Problem | Difficulty |
|---------|-----------|
| Shared memory between instances | Low — QEMU `ivshmem-plain` + file backend |
| Hexagon QEMU has no ivshmem device | **High** — requires patching QEMU Hexagon target |
| Cross-instance interrupt delivery | Medium — need custom virtual device on both sides |
| Hexagon CPU start control from ARM | Medium — no equivalent of `hexagon_globalreg`; needs patch or external coordination |

The core obstacle is that `hexagon-softmmu` has a thin device ecosystem. QBox's SystemC TLM router bridges the two QEMU instances transparently without any QEMU patches.
