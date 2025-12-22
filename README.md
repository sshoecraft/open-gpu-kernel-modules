# NVIDIA driver 570.148.08 with P2P for RTX 3090

This is a patched version of NVIDIA's open GPU kernel modules (570.148.08) that enables peer-to-peer (P2P) GPU communication on consumer RTX 3090 GPUs.

Based on [tinygrad/open-gpu-kernel-modules](https://github.com/tinygrad/open-gpu-kernel-modules).

## What This Enables

- Direct GPU-to-GPU memory transfers via PCIe BAR1
- Works on consumer motherboards (tested: AMD X570, Intel C612/X99)
- Supports multi-GPU ML training workloads (vLLM, etc.)

## Performance

| Configuration | Same-Socket P2P | Cross-Socket P2P |
|---------------|-----------------|------------------|
| Stock driver  | Not supported   | Not supported    |
| This patch    | ~9-11 GB/s      | ~7.6 GB/s        |

## Patches Applied

### 1. Force P2P Type to BAR1
**File:** `src/nvidia/src/kernel/gpu/bif/kernel_bif.c`

Forces P2P to use BAR1 aperture instead of mailbox:
- `NV_REG_STR_CL_FORCE_P2P` = `0x11` (read+write enabled)
- `NV_REG_STR_RM_FORCE_P2P_TYPE` = `BAR1P2P`

### 2. Disable BAR1 Console Preservation
**File:** `src/nvidia/arch/nvalloc/unix/src/osinit.c`

UEFI may map console to GPU BAR1, reserving 4MB at offset 0. This breaks the 512MB alignment required for P2P. Fix: unconditionally disable `bPreserveBar1ConsoleEnabled`.

### 3. Force GPU Coherency
**File:** `src/nvidia/src/kernel/gpu/mem_mgr/uvm_gpu.c`

Forces `uvm_parent_gpu_is_coherent()` to return true, bypassing system bus memory window checks.

### 4. GMMU Aperture Mapping
**File:** `src/nvidia/src/libraries/mmu/gmmu_fmt.c`

Maps `GMMU_APERTURE_PEER` to system memory aperture (`fldAddrSysmem`) since native peer aperture only supports 25-bit addresses (designed for NVLink), not 40-bit BAR1 addresses.

### 5. Force PCIe Relaxed Ordering (NEW)
**File:** `src/nvidia/src/kernel/gpu/bif/kernel_bif.c`

Unconditionally enables PCIe Relaxed Ordering during driver init. Required for dual-socket systems where cross-socket P2P traverses QPI/UPI. Without this, strict memory ordering causes:
- 0.31 GB/s cross-socket throughput (vs 7.6 GB/s with RO)
- vLLM crashes with "illegal memory access"

This eliminates the need for `modprobe nvidia EnablePCIERelaxedOrderingMode=1`.

## How to Build

```bash
# 1. Install base driver
# Download from: https://docs.nvidia.com/datacenter/tesla/tesla-release-notes-570-148-08/index.html

# 2. Build patched modules
make modules -j$(nproc)

# 3. Install
sudo make modules_install
sudo depmod -a

# 4. Reload driver (or reboot)
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
sudo modprobe nvidia
```

## Tested Configurations

- **Single-socket:** AMD X570 + 2x RTX 3090 (11.69 GB/s bidirectional)
- **Dual-socket:** Intel C612 (Z10PE-D8 WS) + 4x RTX 3090, 2x Xeon E5-2680 v4

## Limitations

- P2P only works between identical GPU architectures (RTX 3090 to RTX 3090)
- BAR1 P2P is slower than NVLink but much faster than staging through system RAM
- Requires GPUs with resizable BAR support

## Key Files Reference

| Purpose | File |
|---------|------|
| P2P type forcing | `src/nvidia/src/kernel/gpu/bif/kernel_bif.c` |
| BAR1 console fix | `src/nvidia/arch/nvalloc/unix/src/osinit.c` |
| Aperture mapping | `src/nvidia/src/libraries/mmu/gmmu_fmt.c` |
| GPU ops | `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c` |
