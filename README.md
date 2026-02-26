# rcraid-nvme

AMD RAIDXpert2 `rcraid` driver for modern Linux kernels (6.12 to 7.0+) on AMD Ryzen devices.

<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/1ba563fa-8af3-44ec-aace-c8608ee76f10" />

## The problem

The `rcraid` driver from AMD will not build on kernel ~6.12 and newer.  A recent version (9.3.3.302, February 2026) of `rcraid` distributed by [Lenovo](https://support.lenovo.com/bb/en/downloads/DS579640) includes `raidxpert2-9.3.3_00302-1.x86_64.rpm` and also has this issue.  The second problem is that on AMD APU's (ex: Phoenix, Strix Halo) `rcraid` won't recognize NVMe drives, seemingly due to variables expected in EFI that don't exist on these platforms. A recent kernel (~6.19) is needed to take advantage of the APU for AI/LLM workloads so this bad news if you want the highest possible disk I/O with the ability to dual-boot operating systems and read/write eachother's filesystem.

## The solution

This project extracts the Lenovo RPM, applies a set of patches to work with modern kernels, patches the `rcblob` binary to work with APUs, and repackages with DKMS automation to ensure the module is rebuilt in lockstep with kernel upgrades.  It enables a clean, bootable Linux installation on RAID 0 (Stripe) or RAID 1 (Mirror) volumes created by RAIDXpert2.  For APU owners, this brings the Linux version to feature parity with RAIDXpert2 on Windows.

## Patches applied

| Patch | Purpose |
|-------|---------|
| **APU Enabler** | Patches the v2 blob at file offset `0xce4` (after `endbr64`) to return `0x2a030000`, which maps to feature mask `0x01ff` — enables NVMe RAID 0/1/10 on Ryzen APUs |
| **NVMe VID/DID** | Fixes AMD NVMe vendor/device detection on APU platforms where EFI variables are absent |
| **`register_sysctl_sz`** | Replaces removed `register_sysctl()` API (6.11+) |
| **`ccflags-y`** | Replaces deprecated `EXTRA_CFLAGS` in the Makefile |
| **Timer API** | Handled at DKMS build time — `pre_build.sh` switches `from_timer`/`del_timer_sync` (< 6.13) to `timer_container_of`/`timer_delete_sync` (≥ 6.13) |
| **`bios_param` signature** | Handled at DKMS build time — `pre_build.sh` switches between `block_device *bdev` (< 6.18) and `gendisk *disk` (≥ 6.18) |
| **Pahole workaround** | For kernels ≤ 6.12 — `pre_build.sh` swaps `/usr/bin/pahole` with `/bin/true` before the build; a trap in the DKMS `MAKE` line restores it afterward (even on build failure) |
| **`mk_certs` fixes** | Corrects DER typo, adds Fedora and Debian `sign-file` paths |

## Debian and Fedora Setup for Existing Installations

- Download and install the latest .deb or .rpm from the [releases](https://github.com/DesktopECHO/rcraid-nvme/releases) page.

## Debian and Fedora Setup for "Live System" installers (Boot from RAID)

This guide covers installing Fedora or Debian Linux on RAID arrays using their respective Live CD installers and adding bootable rcraid support.

### Prerequisites

- **RAID Configuration**: Create your RAID 0/1 volume using the RAID configuration utility in BIOS, through RAIDXpert2 on Windows or the RAIDXpert2 Linux binary included with this package.
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

## What gets installed

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

## Development

- **`raidxpert2-install`** — Unified build script that detects RPM/DEB distributions and builds the appropriate package. Extracts the upstream RPM, applies all patches (including v2 blob patch at `0xce4`), writes DKMS metadata, distro-specific initramfs assets, and the shared lifecycle script. Generates the package, installs it, then enters the live-CD flow: polls for the installer's target mount (Anaconda `/mnt/sysroot/` for RPM, Calamares `/tmp/calamares-root-*/` for DEB), copies the package, and chroots in to install it on the new system.
- **`pre_build.sh`** — DKMS `PRE_BUILD` hook; runs before every `dkms build`. Applies kernel-version-gated source patches (timer API ≥ 6.13, bios_param ≥ 6.18) and the pahole workaround (≤ 6.12). Symlinks the v2 blob as `rcblob.x86_64.o`.
- **`rcraid-dkms.sh`** — Shared DKMS lifecycle handler called by both RPM `%post`/`%preun` and DEB `postinst`/`prerm`. Handles `dkms add/build/install`, module loading, and initramfs regeneration (auto-detects dracut vs update-initramfs). Eliminates duplicated post-install logic between the two package formats.
