# Hexagon DSP Initialization on Qualcomm SoCs

## Hardware Boot Flow

Hexagon does not boot independently. It is loaded and released from reset by the ARM application processor:

1. **BootROM → ABL (Android Bootloader)** initializes the SoC and loads all firmware images
2. **PIL (Peripheral Image Loader)** — an ARM-side kernel driver — copies the Hexagon firmware binary (`cdsp.mbn`, `adsp.mbn`, etc.) into a reserved shared DRAM region, verifies its signature, then deasserts the Hexagon reset line
3. **Hexagon executes from its reset vector**, which is the entry point of the loaded firmware image
4. The firmware initializes **QuRT** (Qualcomm Real-Time OS), a proprietary microkernel RTOS
5. QuRT starts the **FastRPC server** and DSP subsystem services, signaling readiness back to ARM via shared memory + interrupt

## QuRT

QuRT is not a general-purpose OS. It provides:
- Thread scheduling and IPC
- Basic memory management (no user/kernel space separation)
- No filesystem, no dynamic linking in the traditional sense

DSP "applications" are `.so` shared libraries pushed from the ARM side via FastRPC at runtime and executed within the QuRT environment.

## Simulating This in QBox

QBox's `qemu_cpu_hexagon` has a `start_powered_off` parameter and an `exec-start-addr` / `start-evb` parameter. This maps directly to the post-PIL state:

- ARM-side simulation loads the Hexagon firmware binary into the shared memory region
- ARM releases Hexagon reset (or QBox starts Hexagon with a pre-configured PC)
- Hexagon begins executing from `exec-start-addr`

There is no need to simulate the full PIL trust chain for functional VP purposes — the firmware can be pre-loaded into the memory model before simulation starts.

## Public Documentation Sources

| Topic | Source |
|---|---|
| PIL / remoteproc driver | Linux kernel `drivers/remoteproc/qcom_*`, `drivers/soc/qcom/pil*` |
| FastRPC protocol | Linux kernel `drivers/misc/fastrpc.c` |
| QuRT API | Qualcomm Hexagon SDK (free registration required) |
| QuRT internals | Not publicly documented |
| Firmware image format (`.mbn`) | Partially documented in open-source PIL code |

The Linux kernel remoteproc driver source is the most useful reference for understanding the ARM↔Hexagon initialization handshake.
