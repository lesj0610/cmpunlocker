# cmpunlocker

Unlock tool for the NVIDIA CMP 170HX (GA100) mining card. Restores full SM compute throughput and unlocked HBM2e memory geometry that are restricted in firmware/OTP configuration.


**[Join our Discord community](https://discord.gg/CdHSakKSFv)** for support and discussions.

---
## Proof of Concept

Below are memory and performance results after applying the unlock:

### Memory Unlock Results

<img alt="memory unlock" src="https://github.com/user-attachments/assets/ae062bd8-e3a7-4e73-b9a4-fbcde53f3c7b" width="100%" style="max-width: 900px;" />

### Performance Benchmarks ([OpenCL-Benchmark](https://github.com/ProjectPhysX/OpenCL-Benchmark))

<img alt="performance benchmarks" src="https://github.com/user-attachments/assets/2501506d-420f-4014-9574-b1bd0290eb60" width="100%" style="max-width: 900px;" />

---

## Requirements

- Linux (x86-64)
- Root access
- NVIDIA CMP 170HX
- **nvidia-open 610.43.0x already installed** (libs + firmware)
- Kernel headers matching the running kernel (`linux-headers-$(uname -r)` / `kernel-devel`)
- Secure Boot disabled (patched modules are unsigned)
- Network access on first install (downloads matching stock `open-gpu-kernel-modules` sources)
- Python 3 (used at build time to select 8GB/10GB geometry)

---

## Install

To install cmpunlocker, run the following command:

```bash
sudo ./install.sh
```

To force a certain memory profile, use the `--profile` option:

```bash
sudo ./install.sh --profile=8gb    # 8GB card → 64GB unlock
sudo ./install.sh --profile=10gb   # 10GB card → 40GB unlock
```

Then perform a cold reboot (full power off, then boot).

## What Gets Unlocked

| Feature | Status |
|---|---|
| Full SM compute throughput (SS0/SS1) | Working ✓ |
| Memory geometry (64GB on 8GB cards, 40GB on 10GB cards) | Working ✓ |
| PCIe Gen 2 speeds | Working ✓ |
| Full BAR1 Size (64GB) | Working ✓ |
| JTAG (Host2Jtag register access) | Working ✓ |
| Persistence across reboot (patched modules) | Working ✓ |
| Full-VRAM stability (unbacked top HBM region excluded) | Working ✓ |
| GPU profiling (CUPTI, Nsight Systems, Nsight Compute HW counters) | Working ✓ |

---

## Uninstall

To uninstall cmpunlocker, run the following command:

```bash
sudo ./uninstall.sh --yes
```

Then perform a cold reboot (full power off, then boot).

## Support & Community

Having issues? Need help? Join our [Discord community](https://discord.gg/CdHSakKSFv) to discuss with other users and get support.

---

## Fork note: late-PMA clamp

This fork carries an extra fix on top of `driver/patches/late-pma.patch`: the unbacked top HBM sliver of the unlocked geometry — the last ~150 MiB with no real memory behind it — is excluded from the PMA (the late-PMA region limit is capped at 62 GiB). Upstream exposes that region to the allocator, so filling VRAM to the very top (e.g. a large matmul or a full KV cache) crashes with `Xid 31 ... FAULT_INFO_TYPE_REGION_VIOLATION` at a bogus address. With the clamp the region is simply dropped, leaving ~63.4 GiB of good, fully usable VRAM; allocating past it returns a clean out-of-memory error instead of a hard fault. (Clamp originally by tlswotj, verified on 3× CMP 170HX.)

---

## Fork note: profiling unlock (CUPTI / Nsight)

`driver/patches/cmp-sku-mask.patch` restores GPU profiling, which stock nvidia-open refuses on CMP parts:

```
CUPTI_ERROR_CMP_DEVICE_NOT_SUPPORTED          # CUPTI, PyTorch profiler
ERR_NVCMPGPU - Profiling is not supported on the NVIDIA Crypto
               Mining Processors (CMP) of the target device 0    # Nsight Compute
```

Nsight Systems does not error at all — it just produces a report with no CUDA kernel data.

The block is a single flag, not a hardware limit. The physical RM (GSP firmware, `gpuGetIsCmpSku_GV100`) reports `isCmpSku = TRUE`; kernel RM caches it in `pGpu->pChipInfo` during `_gpuInitChipInfo()`, and userspace reads it back through `NV2080_CTRL_CMD_GPU_GET_INFO_V2` / `NV2080_CTRL_GPU_INFO_INDEX_CMP_SKU`. CUPTI maps that one value straight onto error 42. Nothing else in the open kernel modules consumes the flag, and the HWPM/perfmon path has no CMP gate of its own, so the patch clears the kernel's copy right where it arrives from GSP.

Expected dmesg output:
```
SEC2_DEBUG: cleared isCmpSku for devId 0x20c2
```

Verified end-to-end on CMP 170HX (driver 610.43.02, CUDA 13.0) — the perf counters are fully functional, the flag was the only thing in the way:

| | Stock | Patched |
|---|---|---|
| `cuptiActivityFlushAll` | `42 CUPTI_ERROR_CMP_DEVICE_NOT_SUPPORTED` | `CUPTI_SUCCESS`, records returned |
| `nsys stats --report cuda_gpu_kern_sum` | *does not contain CUDA kernel data* | full kernel trace with timings |
| `ncu --metrics ...` | `ERR_NVCMPGPU` | counters collected |

Sample `ncu` output on a 4M-element vector-add kernel:

```
smsp__inst_executed.sum      2,097,152 inst      # exact: 131072 warps x 16 inst
dram__bytes.sum                  44.28 Mbyte     # vs 48 MB of traffic issued
sm__cycles_elapsed.avg        43,313.8 cycle
DRAM Frequency 1.72 GHz   SM Frequency 1.13 GHz
Memory Throughput 67.14%  Compute (SM) Throughput 18.19%
```

Note that Nsight Compute additionally needs the driver's own admin restriction lifted — run as root, or load the driver with `NVreg_RestrictProfilingToAdminUsers=0`. That gate is unrelated to CMP and applies to every NVIDIA GPU.

`cuda-gdb` remains blocked: `libcuda` carries a separate CMP check for debugging (*"Debugging is not supported on NVIDIA CMP devices."*) that this patch does not address.
