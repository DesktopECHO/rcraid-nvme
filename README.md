# AMD rcraid Driver Patcher for RHEL 10.x

This repository contains tools to patch and install the AMD RAIDXpert2 (rcraid) driver on modern kernels, where the driver fails to compile due to kernel API changes in the 6.x series.

> **Note:** While this documentation references RHEL, these tools are fully compatible with RHEL derivatives including AlmaLinux, Rocky Linux, and other compatible distributions.

## The Problem 

AMD's official rcraid driver (version 9.3.3) fails to compile on modern 6.x kernels, and the following issues require attention:

1. **NVMe RAID detection** — On Phoenix and newer APUs, the driver fails to identify AMD NVMe controllers when the UEFI variable used for VID/DID mapping is missing, and the binary blob limits functionality when the UEFI `NVME_TRAP_DEVICE` variable is absent.

2. **Missing `vmalloc.h` include** — The header is no longer implicitly included.

3. **SCSI API rename** — `slave_configure` was renamed to `sdev_configure`, and the function signature changed to include a `queue_limits` parameter.

4. **Block queue API changes** — Functions like `blk_queue_max_hw_sectors()` now use the `queue_limits` struct directly.

5. **Sysctl API changes** — `register_sysctl()` requires `register_sysctl_sz()` with explicit size parameter.

6. **Bugs in `mk_certs`** — The AMD signing script has a typo (`-outform DEV` instead of `-outform DER`) and only includes Ubuntu paths, not RHEL paths.

## Repository Contents

```
.
├── README.md                 # This file
├── LICENSE                   # MIT license for scripts (see NOTICE.md for AMD software)
├── NOTICE.md                 # Important licensing info about AMD software
├── patch_and_install.sh      # Simple patcher - patches SDK then runs AMD installer
├── restore_originals.sh      # Restores original SDK files
├── rcraid_manager.sh         # Full-featured management tool with menu
├── check_setup.sh            # Verify prerequisites before installing
├── driver_sdk/               # AMD's original driver SDK
└── RAIDXpert2/               # AMD RAIDXpert2 management utility (optional)
```

## Quick Start

### Prerequisites

```bash
# Install required packages
sudo dnf install kernel-devel-$(uname -r) gcc make elfutils-libelf-devel openssl mokutil

# Verify everything is ready
./check_setup.sh
```

### Option 1: Simple Patch and Install

This patches the AMD SDK files (including the binary blob and NVMe detection) and runs the original AMD installer:

```bash
sudo ./patch_and_install.sh
```

### Option 2: Full Management Tool (Recommended)

For more control, Secure Boot handling, DKMS setup, and RPM/ISO building:

```bash
sudo ./rcraid_manager.sh
```

The manager provides:
- System status display
- Full install (patch, build, sign, install, enroll MOK)
- DKMS setup for automatic rebuilds on kernel updates
- Individual steps (patch, build, sign, install separately)
- Sign already-installed modules
- Build RPM packages (dynamically configured for your kernel)
- Build Driver Update Disk ISOs for fresh installations

## Secure Boot

If Secure Boot is enabled, the module must be signed and the signing key enrolled in MOK (Machine Owner Key).

### Automatic Handling

Both scripts handle Secure Boot automatically:
1. Generate a signing key
2. Sign the module
3. Prompt you to enroll the key with MOK

### MOK Enrollment Process

After running the installer, you'll be prompted to create a MOK password. On next reboot:

1. The MOK Manager (blue screen) will appear
2. Select **"Enroll MOK"**
3. Select **"Continue"**
4. Select **"Yes"**
5. Enter the password you created
6. Select **"Reboot"**

### Manual Signing (if needed)

If you installed via DKMS and the module won't load:

```bash
sudo ./rcraid_manager.sh
# Select option 8: "Sign installed module"
# Select option 9: "Enroll MOK key"
# Reboot and complete MOK enrollment
```

## Verifying Installation

After installation and reboot:

```bash
# Check if module is loaded
lsmod | grep rcraid

# Check for RAID devices
lsblk

# View driver messages
dmesg | grep -i rcraid
```

## DKMS Setup

For automatic rebuilds when you update your kernel:

```bash
sudo ./rcraid_manager.sh
# Select option 3: "Setup DKMS"
```

### EPEL Repository Requirement

DKMS is available from the EPEL (Extra Packages for Enterprise Linux) repository. The script will automatically:

1. Check if CRB (CodeReady Builder) repository is enabled (required for EPEL)
2. Enable CRB if needed
3. Install EPEL repository if not present
4. Install DKMS

For manual installation:
```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install dkms
```

### Weak-Modules (Kernel Compatibility)

RHEL uses a "weak-modules" system that allows kernel modules to work across compatible kernel versions. When you build a module for one kernel version, weak-modules automatically creates symlinks in newer kernel directories if the ABI is compatible.

This means:
- A module built for kernel `6.12.0-124.8.1` may automatically work with `6.12.0-124.13.1`
- The script detects and displays weak-module symlinks in the status display
- You don't always need to rebuild after kernel updates!

The status display shows:
```
Installed:      YES (via weak-modules)
  Symlink:      /lib/modules/6.12.0-124.13.1.el10_0.x86_64/weak-updates/rcraid.ko
  Source:       /lib/modules/6.12.0-124.8.1.el10_0.x86_64/extra/rcraid/rcraid.ko
  Built for:    6.12.0-124.8.1.el10_0.x86_64 (compatible with current)
```

### Preserving Signed Modules

If you have Secure Boot enabled and have already signed and enrolled your module, the DKMS setup will offer to preserve the existing signed module for your current kernel. This prevents the need to re-sign after setting up DKMS.

Future kernel updates will trigger rebuilds. If you have a signing key configured, DKMS will automatically sign new builds using a POST_BUILD hook.

### Fresh Install Workflow

After installing RHEL using a Driver Update Disk:

1. The driver is loaded but not signed with your MOK key
2. Run `rcraid_post_install.sh` from the ISO (or use `rcraid_manager.sh`)
3. Generate a signing key, sign the module, and enroll in MOK
4. Reboot and complete MOK enrollment
5. Set up DKMS to preserve the signed module and enable auto-rebuild

## Building an RPM Package

To create an RPM for distribution or fresh installs:

```bash
sudo ./rcraid_manager.sh
# Select option 13: "Build RPM package"
```

### Kernel Selection

When building an RPM, you can choose which kernel version to target:

- **Current kernel** — Build for the running kernel (default)
- **Other installed kernels** — Build for any kernel with kernel-devel installed
- **Custom kernel** — Manually specify a kernel version

This is useful when:
- Creating a driver disk for installation media that uses a different kernel
- Your system has been updated but you need a driver for the original install kernel
- Building for multiple kernel versions

The script will automatically detect available kernel-devel installations and let you choose.

## Driver Update Disk for Fresh Installs

To install RHEL on a system with AMD RAID where the installer doesn't see the disks:

```bash
sudo ./rcraid_manager.sh
# Select option 14: "Build Driver Update Disk ISO"
```

### ISO Creation

The script uses `xorriso` in native mode to create the ISO. If needed, install manually:

```bash
sudo dnf install xorriso
```

### Using the Driver Disk

The ISO is created with the volume label `OEMDRV`, which Anaconda (the RHEL installer) automatically detects and loads. This means in many cases, simply having the ISO available will auto-load the driver.

**Method 1: Automatic detection (recommended)**
```
# Just boot normally with the ISO accessible - Anaconda finds OEMDRV automatically
```

**Method 2: Explicit boot parameter**
```
inst.dd=hd:LABEL=OEMDRV:/rcraid-9.3.3-<kernel>.iso
```

**Method 3: Interactive**
```
inst.dd
```

> **Note:** The ISO includes a stub `ks.cfg` file to prevent "can't find ks.cfg" warnings that can appear when Anaconda interprets the driver disk as a potential kickstart source. This warning is harmless but the stub file suppresses it for a cleaner installation experience.

## Troubleshooting

### Module fails to load with "Key was rejected by service"

Secure Boot is blocking the unsigned module:

```bash
sudo ./rcraid_manager.sh
# Option 8: Sign installed module
# Option 9: Enroll MOK key
# Reboot and complete enrollment
```

### Build fails with missing header or implicit declaration errors

The patches weren't applied. Run:

```bash
./restore_originals.sh
sudo ./rcraid_manager.sh
# Select option 4 to apply patches, then option 5 to build
```

### Build fails with "sdev_configure" or "queue_limits" errors

Make sure you're using the latest `rcraid_manager.sh` which includes all RHEL 10.x patches.

### Build fails with "register_sysctl" errors

This is fixed in the patches. Ensure patches are applied:

```bash
sudo ./rcraid_manager.sh
# Select option 4 to apply patches
```

### RAID array not detected after successful install

1. Ensure the module is loaded: `lsmod | grep rcraid`
2. Check if AHCI is claiming the devices: `lsmod | grep ahci`
3. Try blacklisting AHCI:

```bash
echo "blacklist ahci" | sudo tee /etc/modprobe.d/blacklist-ahci.conf
sudo dracut -f
sudo reboot
```

### "mokutil --import" fails

Ensure you have the signing key:

```bash
ls -la /var/lib/rccert/certs/
# or
ls -la driver_sdk/certs/
```

### Check setup script reports errors

Run the setup checker to diagnose issues:

```bash
./check_setup.sh
```

This will verify all prerequisites are met before you attempt installation.

## Tested Configurations

| Distribution | Version | Kernel | Status |
|-------------|---------|--------|--------|
| RHEL | 10.0+ | 6.12.0+ | ✅ Working |
| AlmaLinux | 10.0+ | 6.12.0+ | ✅ Working |
| Rocky Linux | 10.0+ | 6.12.0+ | ✅ Expected to work |

## Technical Details

### Patches Applied

#### rc_init.c — vmalloc include

```c
// Added:
#include <linux/vmalloc.h>
```

#### rc_init.c — SCSI API rename

```c
// Before:
.slave_configure = rc_slave_cfg,
static int rc_slave_cfg(struct scsi_device *sdev);

// After:
.sdev_configure = rc_slave_cfg,
static int rc_slave_cfg(struct scsi_device *sdev, struct queue_limits *lim);
```

#### rc_init.c — Block queue API

```c
// Before:
blk_queue_max_hw_sectors(sdev->request_queue, 256);

// After:
lim->max_hw_sectors = 256;
```

#### rc_init.c — Sysctl API

```c
// Before:
rcraid_sysctl_hdr = register_sysctl("rcraid", rcraid_table);

// After:
rcraid_sysctl_hdr = register_sysctl_sz("rcraid", rcraid_table, ARRAY_SIZE(rcraid_table) - 1);
```

#### rcblob.x86_64 — GetLicenseLevel patch

```
Offset 0x12C0: mov eax, 0x2A030000; ret; nop*4
Bytes:  b8 00 00 03 2a c3 90 90 90 90
```

Forces the most permissive license tier (feature flags `0x01FF`), enabling all RAID levels and NVMe RAID support regardless of UEFI variable state.

#### rc_init.c — NVMe VID/DID fallback

```c
// Added after RC_Unmap_VidDid call:
if (hw->orig_vendor_id == 0 && hw->orig_device_id == 0 &&
    id->vendor == 0x1022 && id->device == AMD_NVME_DID) {
    hw->orig_vendor_id = 0x1022;
    hw->orig_device_id = AMD_NVME_DID;
}
```

When the UEFI `NVME_TRAP_DEVICE` variable is absent, `RC_Unmap_VidDid` returns zeros. This fallback detects AMD NVMe controllers (PCI 1022:B000) and sets the correct VID/DID so the driver recognizes them.

#### mk_certs

- Fixed typo: `-outform DEV` → `-outform DER`
- Added RHEL kernel paths: `/usr/src/kernels/$KVERS/scripts/sign-file`

## License & Legal

The patch scripts in this repository are licensed under the **MIT License** — see [LICENSE](LICENSE) for details.

**IMPORTANT:** This repository includes AMD's proprietary driver SDK and utilities. Please read [NOTICE.md](NOTICE.md) for important information about:
- AMD's software licensing terms
- Restrictions on redistribution
- Your obligations when using AMD's software

## Contributing

Issues and pull requests welcome! Please test on your specific configuration before submitting.

## Disclaimer

This is an unofficial community project. AMD does not support, endorse, or take any responsibility for these patches or tools.

The patches modify AMD's proprietary driver source code and binary blob solely to fix compilation issues and enable NVMe RAID functionality on RHEL 10.x kernels.

**USE AT YOUR OWN RISK.**
