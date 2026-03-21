# GPU Simulation in QBox

## Architecture: Same QEMU Instance as ARM CPU

Unlike Hexagon DSP (which requires a separate `QemuInstance`), the GPU is modeled as a **PCI device within the same ARM `QemuInstance`**. This is possible because virtio-gpu is a paravirtualized device, not a separate ISA — it attaches to the ARM PCIe bus like any other peripheral.

## Approach: virtio-gpu (Paravirtualization, Not Functional Emulation)

QBox does not simulate Adreno GPU hardware. Instead it uses QEMU's **virtio-gpu** family, which presents a generic virtual GPU to the guest and forwards work to the host:

| Component | QEMU device type | Backend |
|---|---|---|
| `virtio_gpu_pci` | `virtio-gpu-pci` | Basic 2D display |
| `virtio_gpu_gl_pci` | `virtio-gpu-gl-pci` | OpenGL via virgl → host GPU |
| `virtio_gpu_cl_pci` | `virtio-gpu-cl-pci` | OpenCL → host GPU |
| `virtio_gpu_qnn_pci` | `virtio-gpu-qnn-pci` | QNN inference → host QNN runtime |

The guest sees a generic virtio-gpu device, not an Adreno. The host GPU does the actual work.

## virtio_gpu_qnn_pci

This is a Qualcomm-specific extension targeting AI inference offload via QNN (Qualcomm Neural Network SDK). It adds two capabilities on top of standard virgl:

**1. Blob resources + host-visible shared memory**
```cpp
// 2GB memfd region shared between guest and host (zero-copy tensor transfer)
memory-backend-memfd,id=mem1,size=2048M
get_qemu_dev().set_prop_bool("blob", true);
get_qemu_dev().set_prop_int("hostmem", p_hostmem_mb);
```
Large tensor data is placed directly in shared memory, bypassing the virtio queue copy path.

**2. Context init**
```cpp
get_qemu_dev().set_prop_bool("context_init", true);
```
Allows the guest to create virgl contexts with specific capsets, routing inference requests to the QNN runtime on the host.

**Current status:** The `HAVE_VQNN_RESOURCE_BLOB` compile flag is not defined in the current QEMU fork, meaning the dependent QEMU patches (`virtio-gpu-qnn-pci` device + QNN virgl backend) are not yet merged. This is an in-development Qualcomm feature.

## CPU–GPU Communication (vs. DSP)

| | Hexagon DSP | Adreno GPU |
|---|---|---|
| Kernel driver | `fastrpc.c` (mainline) | `msm-kgsl` (not mainline) |
| Communication | FastRPC — function call semantics | KGSL + ringbuffer — command queue semantics |
| ARM triggers GPU | Writes commands to ringbuffer, rings doorbell register | — |
| GPU signals ARM | Interrupt on completion | — |
| Firmware loading | PIL / remoteproc | PIL / remoteproc (`a6xx_sqe.fw`, `zap.mbn`) |
| QEMU model | Full CPU target (`target/hexagon/`) | None (virtio paravirt only) |
| QBox model | Separate `QemuInstance` | PCI device in ARM `QemuInstance` |

## Cloud Deployment Considerations

On a typical cloud host (x86 + NVIDIA/AMD GPU, or CPU-only), the QNN backend behavior is:

- **`QNN_BACKEND_HTP`** (Hexagon Tensor Processor): fails — no Qualcomm hardware present
- **`QNN_BACKEND_GPU`** (OpenCL): works if host has any OpenCL-capable GPU; fails hard if not — no automatic fallback
- **`QNN_BACKEND_CPU`**: always works on any x86 host

QNN does **not** automatically fall back to a lower backend on initialization failure. The guest application must explicitly handle backend selection. Frameworks like ONNX Runtime with the QNN execution provider can manage this fallback logic.

The intended use case for `virtio_gpu_qnn_pci` in cloud environments is **functional validation and software stack development**, not performance-accurate simulation of Adreno hardware.
