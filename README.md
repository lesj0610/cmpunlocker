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

To also negotiate GPU-to-GPU P2P over BAR1, add `--p2p` (off by default):

```bash
sudo ./install.sh --p2p
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
| GPU-to-GPU P2P over BAR1 (`--p2p`) | Opt-in, host dependent |

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

## Fork note: BAR1 P2P

GSP reports PCIe P2P as unsupported on these boards, so the stock driver refuses
peer-to-peer mappings even when the hardware path exists. `--p2p` adds four
patches that change that:

| Patch | Effect |
|---|---|
| `p2p-caps-force.patch` | Clears `pcieP2PReadCaps` / `pcieP2PWriteCaps` after the GSP RPC so P2P is advertised as available |
| `p2p-bar1.patch` | Selects BAR1 as the P2P connection type instead of the default (mailbox/NVLink) path, and carries the BAR1 peer mapping through the bus, IO VA space and UVM layers |
| `p2p-skip-mailbox-peer-preinit.patch` | Skips the mailbox peer pre-init that has no backing hardware here |
| `p2p-bar1-readcap-override.patch` | Reports the BAR1 read capability the caps query expects |

They are applied after `bar1-resize-unlock.patch` and gated at runtime on the
unlockable device IDs, so other GPUs in the same system are untouched. The
series is kept out of the default install because the outcome depends on the
host, not just the card.

Two things this does **not** do:

- It does not make the host route peer traffic. Cards behind different root
  ports, or a chipset that will not forward peer-to-peer writes, can still fail
  after the driver reports P2P as available. `nvidia-smi topo -p2p r` shows what
  the driver advertises, not what works — confirm with a real peer-to-peer copy.
- It does not help if the BAR1 resize did not take effect. Check that BAR1 is
  actually mapped at the unlocked size (`lspci -vv`, `Region 1`) first; a 64 MiB
  BAR1 leaves nothing for BAR1 P2P to map through.

`verify.sh` reports whether the running build includes the series, reading the
`p2p_bar1` marker written next to the installed modules.
