# NVIDIA 610.57.04 P2P kernel driver — with 48GB modded RTX 4090 support

This repository is a fork of [aikitoria/open-gpu-kernel-modules](https://github.com/aikitoria/open-gpu-kernel-modules)
(`610.57.04-p2p-v2`, driver 610.57.04) that adds **working PCIe P2P on 48GB-modded
RTX 4090 cards** (AD102 with 48GB VRAM but only a 32GiB BAR1 aperture).

## Why this exists

The upstream P2P branch enables CUDA/NCCL peer-to-peer over PCIe by treating the
peer GPU's framebuffer as a region of its BAR1 aperture:

- P2P PTEs encode `peerBAR1base + fbPhysOffset`, which is only valid while the
  peer's **whole framebuffer is identity-mapped into BAR1** (static BAR1).
- Static BAR1 requires `BAR1 >= FB`. Modded 48GB 4090s expose 48GB of VRAM but
  only a 32GiB BAR1 (AD102 hardware limit), so static BAR1 can never be enabled
  and P2P on those cards silently fails or hangs.

## The fix (Method 3 — dynamic BAR1 P2P)

Each peer-shared allocation is mapped **individually** into the remote GPU's
BAR1 at map time (dynamic aperture window), instead of relying on one whole-FB
identity region. The window is IOMMU-mapped into the source GPU and peer PTEs
encode `windowDmaBase + offsetWithinAllocation`, so the allocation may live
anywhere in the 48GB FB while only its window occupies BAR1 space.

Implementation (kernel driver only, no userspace changes):

- `src/nvidia/src/kernel/gpu/bus/arch/hopper/kern_bus_gh100.c` — BAR1-P2P
  capability no longer requires static BAR1; mixed static/dynamic pairs are
  rejected; the per-pair whole-FB IOMMU mapping is created only when static
  BAR1 is enabled on both GPUs.
- `src/nvidia/src/kernel/gpu/bus/p2p_api.c` — whole-FB BAR1 DMA info is only
  fetched when static BAR1 exists (dynamic mode reports none).
- `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c` — dynamic per-allocation BAR1
  windows (`_nvGpuOpsDynBar1*`) wired into the UVM external-allocation
  PTE/phys-addr path, with a BAR1 byte-budget guard; torn down when the duped
  peer handle is freed or the subdevice is destroyed.

This work was ported from the
[`590.48.01-p2p-48g` branch](https://github.com/duanyll/open-gpu-kernel-modules)
by duanyll (design docs there cover methods 1–3 and hardware validation logs).

## Supported configurations

| GPU | P2P path |
| -------- | ---------------------------------------------------------------------- |
| RTX 4090 **48GB modded** (32GiB BAR1 < 48GB FB) | **Dynamic BAR1 P2P** (this branch) |
| RTX 4090 24GB / RTX 5090 / mixed-gen (BAR1 ≥ FB) | Static BAR1 P2P (upstream v2 path, unchanged) |

Verified on: 4× RTX 4090 48GB, EPYC Milan, PCIe Gen4 x16, iommu=pt.

## How to use

Prerequisites: Resizable BAR enabled in BIOS, IOMMU in passthrough mode
(`amd_iommu=on iommu=pt` on the kernel cmdline), NVIDIA 610.57.04 userland
(`NVIDIA-Linux-x86_64-610.57.04.run`).

```sh
# 1. build and install the kernel modules
make modules -j$(nproc)
sudo make modules_install
sudo depmod -a

# 2. replace the running driver (or reboot)
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
sudo modprobe nvidia nvidia_uvm nvidia_modeset nvidia_drm

# 3. check P2P capability
nvidia-smi topo -m          # pairs should be P2P-capable
```

NCCL needs the topology level override to use PCIe P2P on this platform
(GPUs sit behind separate root ports; NCCL defaults to the SHM transport):

```sh
NCCL_P2P_LEVEL=SYS <your distributed app>
```

## Verification results (2026-09-05, 4× RTX 4090 48GB)

- `cudaDeviceCanAccessPeer`: 12/12 ordered pairs accessible.
- P2P data integrity: CE copies and SM kernel peer accesses verified
  byte-exact on all 24 direction/mode combinations (256MiB buffers).
- NCCL all_reduce (4 ranks, `NCCL_P2P_LEVEL=SYS`, "via P2P/CUMEM"):

| size | P2P ON | P2P OFF (SHM) |
| ---- | ------ | ------------- |
| 1 GiB | 63.7 ms / 16.9 GB/s algbw | 454 ms / 2.4 GB/s |
| 64 MiB | 4.05 ms / 16.6 GB/s | 28.6 ms / 2.4 GB/s |
| 1 MiB | 0.139 ms | 0.471 ms |
| 64 KiB | 0.046 ms | 0.056 ms |

## Caveats

- With a translating (non-pt) IOMMU the dynamic windows are not mapped into the
  source GPU's IOVA space; passthrough mode is required (same as upstream).
- Keep all P2P buffers small enough that the sum of their BAR1 windows fits in
  the 32GiB aperture (a budget guard refuses maps past
  `BAR1 − 64MiB`); NCCL transport buffers are tiny and unaffected.
- A heterogeneous node mixing normal (static-BAR1) and 48GB (dynamic) cards
  does **not** advertise BAR1 P2P between the two classes (mixed pairs are
  rejected by design); same-class pairs still work.
- Raw `cudaMemcpyPeer` bandwidth can drop to ~4.5 GB/s on some source GPUs
  depending on allocation layout; NCCL-level throughput is not affected.

## Branches

- `610.57.04-p2p-v2` — this branch: upstream v2 P2P + the 48GB dynamic-BAR1 port.
- Upstream `610.57.04-p2p` / `-v2` — aikitoria's P2P branches (no 48GB support).
- `590.48.01-p2p-48g` (external repo) — duanyll's original 48GB work incl. docs.

## Acknowledgments

- aikitoria's P2P mod (originally geohot's approach), nimlgen's 5090 support.
- duanyll's 48GB dynamic BAR1 P2P work this branch ports.
