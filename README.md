# rcraid-nvme

AMD RAIDXpert2 `rcraid` driver for modern Linux kernels (6.12 to 7.0+) on AMD Ryzen devices.

> Ryzen APU owner runs Linux, gets annoyed at rcraid driver, hacks it into submission.

<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/1ba563fa-8af3-44ec-aace-c8608ee76f10" />

**v2.0** · March 12, 2026 · Based on rcraid 9.3.3.302 (Lenovo distribution, February 2026) · Target platform: MinisForum UM890 Pro (Ryzen 9 8945HS / Strix Halo APU)

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Patches Applied](#patches-applied)
- [Debian and Fedora Setup for Existing Installations](#debian-and-fedora-setup-for-existing-installations)
- [Debian and Fedora Setup for "Live System" Installers](#debian-and-fedora-setup-for-live-system-installers-boot-from-raid)
- [What Gets Installed](#what-gets-installed)
- [Uninstalling](#uninstalling)
- [Technical Deep-Dive](#technical-deep-dive)
  1. [APU Feature Set Is Degraded in Linux](#1-apu-feature-set-is-degraded-in-linux)
  2. [Corporate Ownership Timeline](#2-corporate-ownership-timeline)
  3. [Summary](#3-summary)
  4. [Architecture Overview](#4-architecture-overview)
  5. [OS Interface Contract (RC_HW_OS_)](#5-os-interface-contract-rc_hw_os_)
  6. [Licensing and Feature Gating](#6-licensing-and-feature-gating)
  7. [NVMe Subsystem](#7-nvme-subsystem)
  8. [AHCI / SATA Subsystem](#8-ahci--sata-subsystem)
  9. [RAID Engine and Cache](#9-raid-engine-and-cache)
  10. [Array Management and Transformation](#10-array-management-and-transformation)
  11. [Enclosure, SMART, and Diagnostics](#11-enclosure-smart-and-diagnostics)
  12. [Known Patches and Platform Notes](#12-known-patches-and-platform-notes)
- [Development](#development)
- [Words of Warning](#words-of-warning)

## The Problem

The `rcraid` driver from AMD will not build on kernel ~6.12 and newer. A recent version (9.3.3.302, February 2026) of `rcraid` distributed by [Lenovo](https://support.lenovo.com/bb/en/downloads/DS579640) includes `raidxpert2-9.3.3_00302-1.x86_64.rpm` and also has this issue. The second problem is that on AMD APU's (ex: Phoenix, Strix Halo) `rcraid` won't recognize NVMe drives, seemingly due to variables expected in EFI that don't exist on these platforms. A recent kernel (~6.19) is needed to take advantage of the APU for AI/LLM workloads so this is bad news if you want the highest possible disk I/O with the ability to dual-boot operating systems and read/write each other's filesystems.

## The Solution

This project extracts the Lenovo-supplied RPM, applies a set of patches to work with modern kernels, patches the `rcblob` binary to work with AMD APUs (+some CPUs), and repackages with DKMS automation to ensure the module is rebuilt in lockstep with kernel upgrades. It enables a clean, bootable Linux install on RAID volumes created by RAIDXpert2. This brings the Linux version to feature parity with RAIDXpert2 on Windows.

## Patches Applied

| Patch | Purpose |
|-------|---------|
| **APU Enabler** | Patches the blob (file offset `0xce4` on v2, `0x1310` on v1, after the function entry) to return `0x2a030000` — forces sub-type `0x03` so the driver identifies correctly on Ryzen APUs (proper product name, hardware flags, and logging) |
| **Feature Unlock** | Patches the blob (file offset `0x6017c` on v2, `0x5e959` on v1) — changes a single opcode byte from `AND` (`0x21`) to `CMP` (`0x39`), preventing `RC_CoreFeatureSet` from being narrowed below `0xffff` |
| **NVMe VID/DID** | Fixes AMD NVMe vendor/device detection on APU platforms where EFI variables are absent |
| **`register_sysctl_sz`** | Replaces removed `register_sysctl()` API (6.11+) |
| **`ccflags-y`** | Replaces deprecated `EXTRA_CFLAGS` in the Makefile |
| **Timer API** | Handled at DKMS build time — `pre_build.sh` switches `from_timer`/`del_timer_sync` (< 6.13) to `timer_container_of`/`timer_delete_sync` (≥ 6.13) |
| **`bios_param` signature** | Handled at DKMS build time — `pre_build.sh` switches between `block_device *bdev` (< 6.18) and `gendisk *disk` (≥ 6.18) |
| **Pahole workaround** | For kernels ≤ 6.12 — `pre_build.sh` swaps `/usr/bin/pahole` with `/bin/true` before the build; a trap in the DKMS `MAKE` line restores it afterward (even on build failure) |
| **`mk_certs` fixes** | Corrects DER typo, adds Fedora and Debian `sign-file` paths |
| **Stack canary** (v2 only) | v2 was compiled with `-fstack-protector-strong`, emitting 477 `gs:[0x28]` accesses (134 `MOV` prologue + 343 `SUB` epilogue) that fault on kernels lacking `CONFIG_STACKPROTECTOR_STRONG` (e.g. custom builds on Linux ≥ 6.15 where `fixed_percpu_data` was removed). Each 9-byte sequence is replaced in-place with `XOR reg, reg` + NOP padding. v1 (GCC 4.1.1) was built without stack protection and is unaffected. |
| **Retpoline thunk stubs** (v2 only, conditional) | v2 was compiled with `-mfunction-return=thunk-extern -mindirect-branch=thunk-extern`, requiring `__x86_return_thunk` and `__x86_indirect_thunk_*` symbols. `pre_build.sh` checks `Module.symvers` (fallback: `/proc/kallsyms`) and adds `rc_thunk_stubs.o` to the build only when these symbols are absent (`CONFIG_RETPOLINE=n CONFIG_RETHUNK=n`). On standard Fedora/Ubuntu/Debian kernels the stubs are not compiled in. |

Both `rcblob.x86_64` (v1) and `rcblob_v2.x86_64` (v2) ship with patches 1 and 2 applied; v2 additionally receives the stack canary patch. `raidxpert2-install --blob=v1|v2` (default `v2`) selects which one is actually linked into the built module. See [Section 12](#12-known-patches-and-platform-notes) for full detail.

## Debian and Fedora Setup for Existing Installations

- Download and install the latest .deb or .rpm from the [releases](https://github.com/DesktopECHO/rcraid-nvme/releases) page.

## Debian and Fedora Setup for "Live System" Installers (Boot from RAID)

This guide covers installing Fedora or Debian Linux on RAID arrays using their respective Live CD installers and adding bootable rcraid support.

### Prerequisites

- **RAID Configuration**: Create your RAID volume using the RAID configuration utility in BIOS, through RAIDXpert2 on Windows or the RAIDXpert2 Linux binary included with this package.
- **Live CD**: Fedora Workstation Live ISO or Debian Desktop Live ISO
- **Bootable Media**: USB drive created with the Live ISO

### Step-by-Step Installation

1. **Boot the Live CD and Prepare the Environment**
   - Boot your computer with Debian or Fedora install media
   - Open a terminal in the LiveCD environment
   ```bash
   # Install git if needed (Debian)
   sudo apt update && sudo apt install -y git

   # Clone the project
   git clone https://github.com/DesktopECHO/rcraid-nvme.git
   cd rcraid-nvme
   ```

2. **Run the Installer Script**
   ```bash
   ./raidxpert2-install
   ```
   The script will:
   - Detect your distribution (Fedora or Debian)
   - Install build dependencies
   - Patch the RAIDXpert2 driver for modern kernels
   - Build the appropriate package (RPM or DEB)
   - Install it on the live system

3. **Launch the OS Installer and Configure Installation**
   - **Fedora**: Click "Install to Hard Drive" to start Anaconda
   - **Debian**: Click "Install" to start the installer
   - Follow the normal installation steps
   - When selecting installation destination:
     - Choose "Custom" or "Manual" partitioning
     - Select your RAID volume (appears as a single disk)
     - Configure partitions normally (/, /home, swap, etc.)
   - Complete other installation settings

4. **Complete Installation**
   - Let the installer finish the installation process
   - **Important**: When installation completes, do NOT click "Reboot" yet

5. **Return to Terminal**
   - Go back to the terminal where the script is running
   - Press Enter as prompted
   - The script will detect the installed system and install the driver

6. **Reboot**
   - Click the reboot button
   - Your new installation will boot with RAID support enabled

### Troubleshooting

- If RAID volumes don't appear during installation, ensure a volume is configured and visible in RAIDXpert2.
- Check the live system's kernel version compatibility (6.12+ required)

## What Gets Installed

| Path | Contents |
|------|----------|
| `/usr/src/rcraid-9.3.3/` | Patched source tree, `dkms.conf`, `pre_build.sh`, `rcraid-dkms.sh` |
| `/usr/sbin/RAIDXpert2` | GUI management tool |
| `/usr/sbin/rcadm` | CLI management tool |
| `/opt/raidxpert2/` | AMD documentation (PDF) |
| `/etc/dracut.conf.d/90-rcraid.conf` | Dracut config — forces `rcraid` into initramfs (RPM) |
| `/usr/lib/dracut/modules.d/90rcraid/` | Dracut module — `instmods rcraid` (RPM) |
| `/etc/initramfs-tools/hooks/rcraid` | initramfs-tools hook (DEB) — `manual_add_modules rcraid` |
| `/etc/modules-load.d/rcraid.conf` | Loads `rcraid` at boot (DEB) |

## Uninstalling

**Fedora:**
```bash
sudo dnf remove rcraid-nvme
```

**Debian:**
```bash
sudo apt remove rcraid-nvme  # or purge to also clean the DKMS tree
```

Both packages run a removal script that calls `dkms remove`, unloads the module, and regenerates the initramfs without `rcraid`.

## Technical Deep-Dive

To get the Linux driver working on APUs it was important to be familiar with the decades-old RAIDcore Fulcrum architecture. The sections below are what I was able to learn about the product and the heart of the driver: the binary blob.

### 1. APU Feature Set Is Degraded in Linux

If you own a Ryzen APU and configure NVMe RAID through EFI menus or in Windows, you probably noticed those drives are invisible when you boot Linux. The drives don't even appear as block devices. This is not a bug, it's the consequence of AMD's rcraid driver "ignoring" NVMe drives when running on AMD APUs like Phoenix or Strix Halo.

**What happens at boot.** When the BIOS is set to "RAID" mode (as opposed to AHCI or NVMe mode), the AMD firmware intercepts every NVMe drive during PCI enumeration and re-presents it under AMD's own PCI Vendor:Device ID (`1022:b000`). The original drive identity — for example, a Samsung 990 PRO (`144d:a80a`) — is hidden behind this synthetic AMD device. The EFI variable `NvmeTrapDeviceVar` stores the mapping from the trapped AMD ID back to the real drive, but this variable is only populated on Promontory chipset platforms, not on APU-only systems.

When Linux boots, the kernel's native NVMe driver (`nvme.ko`) does not recognize the AMD `1022:b000` device ID, so it never attaches to the drives. Without a block device driver claiming the hardware, no `/dev/nvmeXnYpZ` nodes are created, and no filesystem — NTFS, ext4, XFS, or otherwise — can be mounted. The drives are effectively invisible to the entire Linux storage stack.

**Where rcraid fits in.** AMD's proprietary kernel module, `rcraid`, is the only driver that knows how to claim the `1022:b000` device IDs and drive NVMe hardware through them. When rcraid loads, it takes ownership of every trapped NVMe device, creates its own internal NVMe controller implementation (168 functions, completely independent of the kernel's `nvme.ko`), and presents the resulting storage through the SCSI/block layer as `/dev/sdX` devices. Any RAID arrays configured in the BIOS appear as single logical `/dev/sdX` volumes.

rcraid exposes drives through its own RAID abstraction, not as raw NVMe namespaces. For example, if you have a single NVMe drive with an NTFS partition, the BIOS RAID firmware traps it as a degraded single-drive array "RAID volume." rcraid then presents it as `/dev/sda`, and ntfs-3g or the kernel's ntfs3 driver can mount the NTFS partition from there.

[↑ Back to top](#rcraid-nvme)

### 2. Corporate Ownership Timeline

The Fulcrum RAID stack has passed through six corporate owners over two and a half decades. Each transition is visible in the source code's copyright headers, internal product code prefixes, and embedded string literals. The table below traces the lineage from the original startup to the current AMD stewardship.

| Years | Owner | Evidence and Notes |
|---|---|---|
| 2000–2004 | RAIDCore, Inc. | Founded as a software RAID startup. Copyright headers in `rc_init.c` read "Copyright (c) 2000–2004, RAIDCore, Inc." The blob still contains "RAIDCoreI", "RAIDCoreH9", and "FATAL Error in RAIDCore stack!" strings. Product codes like "RC-S" (RAIDCore Standard) date from this era. |
| 2005–2006 | Broadcom Corporation | Broadcom acquired RAIDCore. Copyright headers shift to "Copyright (c) 2005–2006, Broadcom Corporation." Product codes "BC48XX", "BCstd", "BCDB", and "BC-UNLICENSED" (Broadcom Controller) originate here. |
| 2006–2008 | Ciprico, Inc. | Broadcom divested the RAID IP to Ciprico. Headers read "Copyright © 2006–2008 Ciprico Inc." Ciprico was a storage solutions company based in Plymouth, MN. |
| 2008–2015 | Dot Hill Systems Corp. | Ciprico merged into Dot Hill. This is the longest ownership period and the most prolific in the codebase — headers span 2008–2015 across multiple files. The license agreement references "DHS" (Dot Hill Systems) and the product code prefix "DHS" (DHS0, DHS10, DHS50, DHS52, DHS60, plus "E" extended variants) dates from this era. Dot Hill was headquartered in Carlsbad, CA. |
| 2015–2017 | Seagate Technology LLC | Seagate acquired Dot Hill in 2015. Copyright headers shift to "Copyright © 2015–2016 Seagate Technology LLC." The blob contains the product name string "SEAGATE_JAGUAR" for a Seagate-branded RAID controller variant. One header contains the typo "Copyright © 2051–2016 Seagate" — a 2015→2051 transposition error that was never fixed. |
| 2017–present | Advanced Micro Devices (AMD) | AMD acquired the RAID IP from Seagate around 2017. Copyright headers read "Copyright © 2017–2025, Advanced Micro Devices, Inc." and "Copyright © 2019–2025" on newer files. The internal source tree root is `fulcrum/` and the codebase was rebranded as "RAIDXpert2." AMD ships this as part of their Promontory chipset and APU RAID solution. |

The cumulative effect of six owners is visible throughout the codebase: product code prefixes span RC- (RAIDCore), BC- (Broadcom), DHS- (Dot Hill), and SEAGATE_ (Seagate). The license boilerplate still references "DHS" by name. Debug strings like "FATAL Error in RAIDCore stack!" coexist with AMD copyright notices. This archaeological layering is important context for anyone maintaining the code — naming conventions and internal terminology shift depending on which era introduced a given subsystem.

[↑ Back to top](#rcraid-nvme)

### 3. Summary

The AMD RAIDXpert2 kernel module (`rcraid`) is built on the "Fulcrum" RAID stack — a portable storage engine compiled from 54 C++17/C source files into a single ELF relocatable object (the "blob"). The blob links at build time with an open-source C wrapper that provides Linux kernel integration: PCI probing, DMA mapping, interrupt handling, and SCSI host registration. As explained in [Section 1](#1-apu-feature-set-is-degraded-in-linux), rcraid is the sole driver capable of claiming NVMe drives that the BIOS has trapped behind AMD's synthetic `1022:b000` PCI ID in RAID mode — without it, those drives (and any NTFS or other partitions on them) are invisible to Linux.

Fulcrum implements RAID levels 0, 1, 5, 6, 10, 50, 60, Volume (JBOD), and RAIDABLE (JBOD-to-RAID live conversion). It includes online transformation (live RAID level migration, capacity expansion), hot spare management (global, dedicated, distributed), write-back caching with battery/NVRAM persistence, SMART monitoring, TRIM passthrough, tiered storage, SGPIO enclosure LED control, native 4K sector support, AVX2-accelerated parity, and a complete NVMe controller implementation (168 `NVM_` symbols). The same blob handles hardware from discrete PCI RAID cards with battery-backed cache down to consumer APUs with nothing but a PCIe BAR.

**Key statistics** (v2, `rcblob_v2.x86_64`):

| Property | Value |
|---|---|
| File size | 12,383,448 bytes (11.8 MB) |
| Executable code (.text) | 633,262 bytes (618 KB) |
| Read-only data (.rodata) | 1,166,272 bytes (1.1 MB) |
| BSS (uninitialized data) | 431,096 bytes (421 KB) |
| Functions | 1,580 (1,515 global, 61 local, 4 weak) |
| Relocations | 192,886 (75.5% R_X86_64_32) |
| State machine states | 1,180 (RC_CORE_STATE_) |
| Management commands | 82 (RC_CMD_) |
| Event types | 145 (RC_MSGID_), ~20 languages |
| OS callbacks | 56 (RC_HW_OS_) |
| Compiler | GCC 13.3.0, C++17, kernel model |

[↑ Back to top](#rcraid-nvme)

### 4. Architecture Overview

The Fulcrum stack is organized in four layers. The **Platform layer** (`rc_sth_*` files, 7 .cpp + 1 .c) translates between the Linux SCSI subsystem and Fulcrum's internal I/O model, handling initialization sequencing, timer management, and the `RC_OurInterfaceStruct` (80 bytes). The **Core layer** (21 .cpp files in `fulcrum/rc/core/`) implements RAID logic: array creation, configuration, generic/bypass/direct I/O dispatch, caching, tiered storage, online transformation, and RAID 5/6 I/O. The **Bottom layer** (protocol drivers in `fulcrum/rc/drivers/`) provides AHCI/SATA, NVMe, SAS, SCSI, SGPIO, port multiplier, native 4K, and SES support. The **Shared layer** holds utilities, feature key validation, CMOS/flash memory management, and the CHID manager.

**Binary structure.** The blob (`rcblob_v2.x86_64`) is an ELF relocatable with 47 sections. The main `.text` section (633 KB, 1,580 functions) starts at file offset `0x90`. Read-only data (`.rodata`, 1.1 MB) holds jump tables, the message catalog, and the ProductDescTable. The `.data` section (42 KB) contains initialized globals including `NVM_Vendor_` strings, LED descriptors, and the feature set variable. The `.bss` section (421 KB) holds queue blocks, hash tables, and statistics counters. DWARF debug info (2.2 MB) preserves full source paths under the `fulcrum/` tree.

**Compilation flags:**

```
GNU C++17 13.3.0 -mno-red-zone -mcmodel=kernel -mno-sse -m64
-mindirect-branch=thunk-extern -mfunction-return=thunk-extern
-mharden-sls=all -fstack-protector-strong -fcf-protection=branch
-mrecord-mcount -mfentry -O2 -fno-exceptions -fno-strict-aliasing
```

Notable hardening: CET/IBT landing pads (`endbr64`), retpoline indirect branches, SLS mitigation, and stack canaries. The `-mrecord-mcount -mfentry` flags enable ftrace probe attachment to blob functions at runtime. The 192,886 relocations (75.5% `R_X86_64_32`) are the primary engineering constraint for binary patching.

**Kernel compatibility caveats (v2).** Two compilation flags create problems on certain kernel configurations. (1) `-fstack-protector-strong` emits 477 `gs:[0x28]` accesses; on kernels built without `CONFIG_STACKPROTECTOR_STRONG` (e.g. custom builds on Linux ≥ 6.15 where `fixed_percpu_data` was removed), that address is unmapped and the first call into the blob faults. The stack canary patch neutralizes all 477 sites unconditionally. (2) `-mfunction-return=thunk-extern` and `-mindirect-branch=thunk-extern` require `__x86_return_thunk` / `__x86_indirect_thunk_*` symbols absent on `CONFIG_RETPOLINE=n CONFIG_RETHUNK=n` kernels; `rc_thunk_stubs.S` is compiled in conditionally. v1 (GCC 4.1.1, 2006) predates both features and is unaffected. See [Patches Applied](#patches-applied) and [Section 12](#12-known-patches-and-platform-notes).

**v1 vs v2.** AMD ships two builds of the same Fulcrum source: `rcblob.x86_64` (v1) and `rcblob_v2.x86_64` (v2, described above). The upstream Makefile picks v2 only when `$(KVERS)` matches `*el10*` (RHEL 10) and falls back to v1 for every other kernel string. This project applies patches 1 and 2 to both blobs (v2 also gets patch 3, the stack-canary neutralization) and lets `raidxpert2-install --blob=v1|v2` (default v2) choose which one is actually linked into the module.

| | v1 (`rcblob.x86_64`) | v2 (`rcblob_v2.x86_64`) |
|---|---|---|
| Compiler | GCC 4.1.1 (2006) | GCC 13.3.0 |
| File size | 10,601,800 bytes | 12,383,448 bytes |
| ELF sections | 34 | 47 — adds `.note.gnu.property` and per-function COMDAT `.group` sections |
| Hardening | None — no CET note, no retpolines, no stack protector | CET/IBT (`endbr64`), retpolines, `-mharden-sls=all`, `-fstack-protector-strong` |
| Debug format | Pre-DWARF5 (`.debug_pubnames`, `.debug_loc`, `.debug_ranges`) | DWARF5 (`.debug_loclists`, `.debug_rnglists`, `.debug_line_str`) |
| Global functions | 1,520 | 1,515 — 5 fewer as globals; folded into weak COMDAT sections instead (same code, just different linkage) |

Functionally the two are identical: the same 7 `NVM_` NVMe symbols and the same 82 `RC_CMD_` management commands, byte-for-byte, exist in both. The differences are entirely toolchain and hardening generation, not RAID features — which is also why patches 1 and 2 apply to both blobs (the same instructions exist in each, just at different file offsets; see [Section 12](#12-known-patches-and-platform-notes)).

[↑ Back to top](#rcraid-nvme)

### 5. OS Interface Contract (RC_HW_OS_)

The blob communicates with the wrapper through 56 `RC_HW_OS_` callbacks organized into functional categories. These form the only seam between the proprietary RAID engine and the Linux kernel.

| Category | Count | Key Functions |
|---|---|---|
| Memory/DMA | 7 | MapIoSpace, GetDmaAddress, GetPhysicalAddress, GetVirtualAddress |
| PCI Config | 10 | ReadPCI{Byte,Word,Dword}, WritePCI{Byte,Word,Dword}, ReadPCIIO, WritePCIIO |
| Atomics | 4 | AtomicAND/OR/Exchange/SetValue |
| EFI/NVRAM | 1 | ReadNvramVar (NvmeTrapDeviceVar, etc.) |
| DPC/Timer | 5 | InitializeDpc, ScheduleDpc, DpcRoutine, DelayMicroSeconds, ReadPerfCycleCounter |
| Power/ODD | 4 | PowerDown/UpODD, CheckODDPower, EjectPending |
| Licensing | 1 | GetLicenseLevel (CPUID-based, Patch 1 target) |
| Debug | 2 | DebugPrintf, LevelDebugPrintf |
| Info/Config | 13 | MaxCPUMemory, MaxStructureSize, ProductIndex, IocAllocSize, etc. |
| SCSI/Dump | 5 | ReadyForNextRequest, CopyToUserMem, DiskDumpActive, HiberDump |
| ACPI | 3 | HandleAcpiNotification, GetPortInitCommands, SaveGTFRegisters |
| Misc | 1 | GetSupport4kNativeDisks / SetSupport4kNativeDisks / UsePio |

The blob calls `RC_Initialize()` at load time, which populates `RC_OurInterfaceStruct` (80 bytes in `.data`) with function pointers to its internal entry points. The wrapper then calls through this struct to drive the blob's state machines.

[↑ Back to top](#rcraid-nvme)

### 6. Licensing and Feature Gating

The feature gating pipeline has four stages: (1) `RC_HW_OS_GetLicenseLevel` executes CPUID and maps processor family/model/stepping to a sub-type. (2) `RC_Proto_DiscoverAdapter` converts the sub-type to a 16-bit per-controller feature mask via a lookup table. (3) `RC_ScanBus` initializes the global `RC_CoreFeatureSet` to `0xFFFF`, then AND-reduces it with each controller's mask. (4) Every array operation tests specific bits of `RC_CoreFeatureSet` before proceeding.

APU processors (Phoenix, Strix Halo, etc.) map to default sub-type `0x14` with mask `0x0400` (management passthrough only, no RAID). Sub-type `0x03` is the "RC-S" profile with a hardcoded fast-path yielding mask `0x01FF` (RAID 0/1/5/10). When a feature check fails, the blob returns status `0x38` (`RC_STS_NO_SW_LICENSE`), which appears 204 times across ~20 languages in the message catalog.

**Feature bits:**

| Bit | Feature | Enforcement |
|---|---|---|
| 0–1 | RAID 0, RAID 1 | RC_CreateRaidArray |
| 2, 7 | RAID 10 | RC_CreateRaidArray, RC_CreateTransformRaidArray |
| 3–4 | Volume, RAIDABLE | RC_CreateRaidArray |
| 5 | RAID 5 | RC_CreateTransformRaidArray |
| 6 | Controller spanning | RC_CreateTransformRaidArray |
| 8 | Online expansion/migration | RC_CreateTransformRaidArray |
| 9 | RAID 6 | RC_CreateTransformRaidArray |
| 10 | Management passthrough | Various |

For discrete RAID cards, hardware licensing uses 1-Wire EEPROMs (Dallas DS2502). On APU/integrated platforms there is no EEPROM, so validation fails and the mask stays restrictive, hence the need for the CPUID patch to sidestep the entire key validation path.

**Product Description Table.** `RC_ProductDescTable` (96 bytes in `.rodata`) maps VID:DID:CC combinations to product profiles. Six entries cover AMD NVMe (`1022:b000`), Bristol (`1022:7905`), Summit (`1022:7916`), Summit SP3 (`1022:7917`), Promontory (`1022:43bd`), and RAM+AMD (`8590:1022`). Product code prefixes trace the ownership timeline: RC-S (RAIDCore), BC48XX/BCstd (Broadcom), DHS0/DHS10/DHS50/DHS60 (Dot Hill), SEAGATE_JAGUAR (Seagate), and NVME_RAID (AMD).

[↑ Back to top](#rcraid-nvme)

### 7. NVMe Subsystem

The blob contains a complete NVMe controller implementation (168 `NVM_` symbols) that does not wrap the kernel's nvme driver. It directly manages NVMe queues: creates admin + I/O queue pairs, submits commands via doorbell writes, and processes completions via MSI-X interrupts. The implementation covers the full NVMe 1.3+ command set including Identify Controller/Namespace, Read/Write, Flush, DSM (TRIM/Deallocate), Abort, Asynchronous Event Request, Get/Set Features, Get Log Page, Format NVM, Firmware Download/Commit, and Namespace Management.

**Vendor-specific handling.** `nvmStartIo` contains one vendor dispatch point. It reads the PCI Vendor ID from the NVMAdapter structure at offset `+0x7a0` and compares against two known VIDs: `0x1bb1` (Seagate) and `0x144d` (Samsung). Samsung drives get `NVM_Vendor_Samsung` ("SAMSUNG "); Seagate drives get `NVM_Vendor_Seagate` ("SEAGATE "); all others get `NVM_Vendor_Unknown` ("UNKNOWN "). This classification is purely cosmetic for the RAIDXpert2 management GUI. There are no vendor-specific NVMe command quirks, TRIM workarounds, or firmware-specific code paths — all NVMe drives are handled identically at the I/O level.

On Promontory platforms, the EFI variable `NvmeTrapDeviceVar` resolves the trapped AMD VID:DID (`1022:b000`) back to the original drive vendor (e.g., Samsung `144d:xxxx`). On APU platforms without this EFI variable, both `orig_vendor_id` and `orig_device_id` are zero, and the vendor falls through to "UNKNOWN."

[↑ Back to top](#rcraid-nvme)

### 8. AHCI / SATA Subsystem

The AHCI driver (`rc_hw_ahci.cpp` and related files) supports five AMD SATA controller families: Bristol Ridge (`7905`), Summit Ridge (`7916`), Summit SP3 (`7917`), Promontory (`43bd`), and legacy SP5100 (`4392`/`4393`) and Bolton (`7802`/`7803`). It directly programs the AHCI Host Bus Adapter registers (GHC, PxCLB, PxFB, PxCI, PxSERR) and builds command tables with scatter-gather lists.

Additional SATA features include FIS-based port multiplier support (`rc_hw_ahci_pm.cpp`) for drive-bay expanders, ZPODD (Zero-Power Optical Disc Drive) management via ACPI methods (PRID/SECD/ODD), asynchronous notification for hot-plug events (`rc_hw_ahci_an.cpp`), and SGPIO LED control for enclosure management (`rc_hw_sgpio.cpp`). SMART polling runs on a configurable interval (`RC_SmartPollInterval` in `.data`) and the SMART log handler (`RC_HW_SATA_CopySmartLog`) parses standard SMART attributes plus three vendor-specific pages.

[↑ Back to top](#rcraid-nvme)

### 9. RAID Engine and Cache

**RAID levels.** Fulcrum supports RAID 0, 1, 5, 6, 10, 50, 60, Volume (JBOD passthrough), and RAIDABLE (JBOD-to-RAID live conversion). Three I/O paths exist: the **Bypass path** (`RC_StartBypassIo`) handles simple RAID levels (0, 1, JBOD) with a streamlined state machine that skips the full generic pipeline. The **Generic path** (`rc_generic_io.cpp`) handles complex RAID levels through the full 1,180-state machine. The **Direct path** (`rc_direct_io.cpp`) bypasses the RAID layer entirely for physical disk management operations.

**Parity engine.** RAID 6 uses an even-odd parity algorithm (`rc_raid6_evenodd.cpp`) with the `memxor_many` function (7,587 bytes, the largest function in the blob). AVX2 acceleration via `rcfx_*_avx` functions is gated behind the `RC_SupportAVX` flag despite the global `-mno-sse` compilation flags — the kernel saves/restores FPU context around these sections. Native 4K sector handling (`rc_native4k.cpp`) implements a full read-modify-write pipeline for 4K-native drives behind 512-byte RAID stripe boundaries.

**Write-back cache.** The cache subsystem (`rc_cache.cpp`) implements enterprise-grade write-back caching with an ARC (Adaptive Replacement Cache) variant using a ghost cache for frequency tracking. For discrete controllers with battery backup or NVRAM, dirty data persists across power failures via `RC_CacheFlush` and dirty tag recovery. Runtime tuning parameters include `RC_CacheLineSize`, `RC_CacheReadAhead`, `RC_DoWriteBack`, `RC_CacheFlushTimer`, and `RC_CacheMaxDirtyPercent`. On consumer APU platforms, write-back mode is typically disabled since there is no non-volatile backing store.

[↑ Back to top](#rcraid-nvme)

### 10. Array Management and Transformation

The online transformation engine (`rc_transform.cpp`, `rc_transform_drivers.cpp`) supports live RAID level migration (e.g., RAID 1 to RAID 5), capacity expansion (adding drives to an existing array), and stripe-size changes — all without taking the array offline. The transformation process uses a cursor that tracks the conversion boundary; I/Os below the cursor go to the new layout, I/Os above go to the old layout, and boundary-spanning I/Os are split.

Rebuilds use a QSync bitmap (`rc_qsync_bitmap.cpp`) that tracks which regions need resynchronization at sub-stripe granularity, allowing targeted rebuilds after brief outages rather than full-array rebuilds. Hot spare management supports three modes: global (any array), dedicated (specific array), and distributed (spare capacity spread across existing drives, with SMART pre-screening for candidate selection).

Additional array operations: mirror split/copy (detaching a mirror half as a snapshot), secure erase (ATA SECURITY ERASE UNIT and NVMe Format with secure erase), TCG Opal support for self-encrypting drives, and third-party metadata detection (`RC_ReadThirdPartyMetaData` recognizes legacy Promise RAID arrays but forces them offline rather than importing).

[↑ Back to top](#rcraid-nvme)

### 11. Enclosure, SMART, and Diagnostics

**Enclosure management.** The SGPIO subsystem (`rc_hw_sgpio.cpp`) drives enclosure LEDs through a state machine with per-drive LED states for activity, locate, and fault. SAS expander support includes zone locking and SES (SCSI Enclosure Services) device management (`rc_sep.cpp`, `rc_ses.cpp`). A buzzer/alarm system generates audible alerts on controller events.

**SMART and health monitoring.** SMART polling runs on a configurable interval for SATA drives (`RC_SmartPollInterval`). NVMe health monitoring uses the Get Log Page command for SMART/Health Information (Log ID `0x02`). The health check function (`RC_CheckDeviceListSmartErr`) scans the device list and raises `RC_MSGID_` events when thresholds are exceeded. Three vendor-specific SMART pages are parsed (`RC_SW_SmartData_VendorSpecific1/2/3`).

**Debug and fault injection.** The blob includes a comprehensive fault injection framework: `RC_CMD_INJECT_ERROR`, `SET_FORCE_ERROR`, `SET_FORCE_BBR_ERROR`, and `SET_FORCE_MEM_ERROR` allow triggering specific failure modes for testing. An event catalog with 145 `RC_MSGID_` types covers all operational events, localized across approximately 20 languages in a 39,664-byte message lookup table. The 82 `RC_CMD_` management commands provide the full configuration/management API accessible through the RAIDXpert2 GUI and `rcadm` CLI.

[↑ Back to top](#rcraid-nvme)

### 12. Known Patches and Platform Notes

**Binary blob patches.** Two patches to the blob are required for APU platforms that lack a Promontory chipset. Both `rcblob.x86_64` (v1) and `rcblob_v2.x86_64` (v2, see [Section 4](#4-architecture-overview)) get patches 1 and 2; v2 additionally gets patch 3. `raidxpert2-install --blob=v1|v2` (default v2) selects which patched blob is actually linked into the module.

| Patch | v1 Offset | v2 Offset | Change | Purpose |
|---|---|---|---|---|
| Patch 1 (CPUID) | `0x1310` | `0xCE4` | Overwrite the start of `RC_HW_OS_GetLicenseLevel` with `mov eax, 0x2a030000 ; ret`. v2 preserves a 4-byte `endbr64` CET landing pad before the overwrite; v1 has none (GCC 4.1.1 predates CET), so the patch starts right at the function entry. | Forces the CPUID identity check to always return sub-type `0x03` (RC-S, RAID enabled) instead of the restrictive `0x14` (APU, no RAID) |
| Patch 2 (Feature Gate) | `0x5E959` | `0x6017C` | Single opcode byte in `RC_ScanBus`: `and [RC_CoreFeatureSet], eax` (`0x21`) → `cmp [RC_CoreFeatureSet], eax` (`0x39`). The ModRM byte, displacement, and `R_X86_64_PC32` relocation are all unchanged, so the module loader stays happy. | Converts the AND-reduction (which narrows the feature mask) into a read-only comparison, leaving `RC_CoreFeatureSet` at `0xFFFF` and enabling all RAID levels |
| Patch 3 (Stack Canary) | — | whole blob | Scans all 477 `gs:[0x28]` accesses (9-byte `65 4W {8b,2b} MM 25 28 00 00 00` sequences) and replaces each in-place with `XOR reg, reg` + NOP padding. `MOV` prologue → 32-bit XOR (zero-extends; saved canary = 0). `SUB` epilogue → 64-bit XOR (diff = 0; check always passes). No relocations touched. | v2 compiled with `-fstack-protector-strong` faults on kernels lacking `CONFIG_STACKPROTECTOR_STRONG` (≥ 6.15 where `fixed_percpu_data` was removed). v1 needs no patch. |
| Retpoline thunk stubs | — | — (conditional) | `rc_thunk_stubs.S` compiled in at DKMS build time only when `__x86_return_thunk` is absent from `Module.symvers` / `/proc/kallsyms`. Provides direct `ret; int3` and `jmp *%reg` fallbacks. v1 predates retpolines and is unaffected. | v2 compiled with `-mfunction-return=thunk-extern -mindirect-branch=thunk-extern`; linking fails on `CONFIG_RETPOLINE=n CONFIG_RETHUNK=n` kernels without these stubs |

**C source patches.** The open-source wrapper requires patches for kernel compatibility and NVMe support: adding the NVMe PCI class and `AMD_NVME_DID` (`0xb000`) to `rcraid_id_tbl[]`, implementing `RC_Unmap_VidDid()` for EFI variable lookup, kernel API adaptations (`class_create` signature changes, `blk_alloc_disk` API, timer API, `access_ok` signature), and DKMS `pre_build` hooks that use `kver_ge()` to conditionally apply kernel-version-specific patches.

**Platform notes:**

| Platform | Characteristics |
|---|---|
| Ryzen APU (no Promontory) | No `NvmeTrapDeviceVar` EFI variable; NVMe drives appear as AMD `1022:b000` (invisible to Linux without rcraid — see [Section 1](#1-apu-feature-set-is-degraded-in-linux)); Patches 1 and 2 required; CPUID maps to restrictive sub-type without Patch 1; no hardware RAID key (EEPROM absent); vendor shows as "UNKNOWN" in management UI. Tested on MinisForum UM890 Pro (Ryzen 9 8945HS). |
| Promontory Chipset | EFI firmware populates `NvmeTrapDeviceVar` with original drive VIDs; the wrapper's `RC_Unmap_VidDid` resolves trapped devices. Patches 1 and 2 are still applied (the blob is patched at install time regardless of platform), but the CPUID sub-type mapping may already permit RAID here without Patch 1. |
| Discrete RAID Cards | Full hardware support: battery/NVRAM for write-back cache, 1-Wire EEPROM for licensing, SGPIO for enclosure LEDs, SAS expanders, buzzer/alarm. No binary patches needed — hardware license key enables features natively |

[↑ Back to top](#rcraid-nvme)

## Development

- **`raidxpert2-install`** — Unified build script that detects RPM/DEB distributions and builds the appropriate package. Extracts the upstream RPM, patches both the v1 and v2 blobs (`--blob=v1|v2` selects which is linked in, default v2), writes DKMS metadata, distro-specific initramfs assets, and the shared lifecycle script. Generates the package, installs it, then enters the live-CD flow: polls for the installer's target mount (Anaconda `/mnt/sysroot/` for RPM, Calamares `/tmp/calamares-root-*/` for DEB), copies the package, and chroots in to install it on the new system.
- **`pre_build.sh`** — DKMS `PRE_BUILD` hook; runs before every `dkms build`. Applies kernel-version-gated source patches (timer API ≥ 6.13, bios_param ≥ 6.18) and the pahole workaround (≤ 6.12), and links `rc_thunk_stubs.o` into the module only when the kernel does not export `__x86_return_thunk` (`CONFIG_RETPOLINE=n CONFIG_RETHUNK=n`). Symlinks the selected blob as `rcblob.x86_64.o`.
- **`rcraid-dkms.sh`** — Shared DKMS lifecycle handler called by both RPM `%post`/`%preun` and DEB `postinst`/`prerm`. Handles `dkms add/build/install`, module loading, and initramfs regeneration (auto-detects dracut vs update-initramfs). Eliminates duplicated post-install logic between the two package formats.

## Words of Warning

While the original RPM package contains only AMD-authored code, it has been patched from its original form and therefore not supported by AMD. **Use at your own risk.** Always back up important data before installing kernel drivers from unofficial sources.
