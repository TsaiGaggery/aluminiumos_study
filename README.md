# GRUB/ESP Boot Support for AluminiumOS

This repository contains patches, scripts, and prebuilts to boot AluminiumOS (Android Desktop) on Intel platforms using UEFI/GRUB instead of ChromeOS firmware.

## Supported Targets

| Target | Platform | Description |
|--------|----------|-------------|
| **fatcat** | Panther Lake (PTL) | Primary target for PTL hardware |
| **firefly** | Wildcat Lake (WCL) | WCL variant without ChromeOS EC |
| **ocelot** | Wildcat Lake (WCL) | Original WCL target |

**Note**: fatcat and firefly patches are based on **WW02 BKC** (Build Known Configuration).

## Quick Start

```bash
# Clone to Android source root
cd $ANDROID_BUILD_TOP
git clone <repo_url> alos_grub

# Deploy patches and prebuilts for fatcat
./alos_grub/deploy.sh --target=fatcat

# Or for firefly (WCL without EC)
./alos_grub/deploy.sh --target=firefly

# Build
source build/envsetup.sh
lunch fatcat-userdebug   # or firefly-userdebug
m -j
```

## Contents

```
alos_grub/
├── README.md                      # This file
├── CLAUDE.md                      # AI assistant context
├── deploy.sh                      # Deployment script
├── cleanup_patches.sh             # Remove applied patches by author
├── build_prebuilt_esp.sh          # Script to rebuild prebuilt ESP image
├── patches/                       # Patches for Android repos
│   ├── build/make/
│   ├── device/google/desktop/common/
│   ├── device/google/desktop/fatcat/
│   ├── device/google/desktop/ocelot/
│   ├── frameworks/base/cmds/bootanimation/
│   └── vendor/google/desktop/
├── new_files/                     # New files to copy (not patches)
│   └── device/google/desktop/firefly/   # Firefly device config
├── installer/                     # Installer scripts
│   └── android-desktop-install   # Dual-boot installer backup
├── kernel_patches/                # Kernel patches (manual application)
│   ├── README.md                  # Application instructions
│   └── *.patch
└── prebuilts/                     # Prebuilt binaries
    ├── esp.img                    # 64MB FAT32 ESP with GRUB + theme
    ├── BOOTX64.EFI                # GRUB EFI bootloader
    ├── boot_splash.png            # Boot splash (logo centered on black)
    ├── fonts/unicode.pf2          # Font for graphics menu
    ├── theme/                     # GRUB theme files
    │   ├── theme.txt
    │   └── background.png
    ├── gfxmenu.mod                # Theme support module
    ├── trig.mod                   # Theme support module
    └── *.mod                      # Other GRUB modules
```

## Deploy Options

```bash
# Deploy for fatcat (Panther Lake) - copies to aosp_diff and prebuilts
./alos_grub/deploy.sh --target=fatcat

# Deploy for firefly (Wildcat Lake without EC)
# Clones aosp_diff/fatcat to aosp_diff/firefly, then adds firefly patches
./alos_grub/deploy.sh --target=firefly

# Deploy for ocelot (original Wildcat Lake)
./alos_grub/deploy.sh --target=ocelot

# Copy patches to aosp_diff tree only
./alos_grub/deploy.sh --target=fatcat --aosp-diff

# Apply patches directly to git repos
./alos_grub/deploy.sh --target=fatcat --apply

# Copy prebuilts only
./alos_grub/deploy.sh --target=fatcat --prebuilts

# Remove patches by author (cleanup)
./alos_grub/deploy.sh --target=fatcat --clean

# Or use standalone cleanup script
./alos_grub/cleanup_patches.sh [--dry-run] [ANDROID_ROOT]

# Dry run - see what would happen
./alos_grub/deploy.sh --target=fatcat --dry-run --all
```

## Building the GRUB Prebuilts

The prebuilts can be regenerated using the included script:

```bash
# Prerequisites (Ubuntu/Debian)
sudo apt install grub-efi-amd64-bin grub-common mtools dosfstools

# Build the prebuilt ESP image
./build_prebuilt_esp.sh
```

This creates `prebuilts/esp.img` containing:
- GRUB EFI bootloader (BOOTX64.EFI)
- GRUB theme with centered logo
- GRUB modules for graphics and theme support
- Unicode font for graphics menu
- Boot splash image for post-menu display

## Kernel Patches

Kernel patches for `alos-kernel-6.18` are in `kernel_patches/`. These are NOT automatically applied - see `kernel_patches/README.md` for manual application instructions.

Features enabled:
- BGRT (Boot Graphics Resource Table) support
- Intel MEI GSC modules
- Realtek RTW88 USB WiFi drivers

## What Each Patch Does

### Common Patches (device/google/desktop/common/)

| Patch | Purpose |
|-------|---------|
| `0001-Enable-GRUB-ESP-building` | Includes esp_image.mk in board_x86.mk |
| `0002-Make-GSC-packages-conditional` | DESKTOP_DISABLE_GSC flag for GSC packages |
| `0003-Make-TrustyVM-conditional` | DESKTOP_DISABLE_TRUSTY_VM flag |
| `0004-Make-headless-system-user-conditional` | DESKTOP_DISABLE_HSUM flag |
| `0005-Make-EC-packages-conditional` | DESKTOP_DISABLE_EC flag |
| `0006-Remove-unsupported-fbsplash` | Removes fbsplash feature |
| `0007-Add-SELinux-policy-for-BGRT` | SELinux policy for /sys/firmware/acpi/bgrt |

### Fatcat Patches (device/google/desktop/fatcat/)

Based on **WW02 BKC**.

| Patch | Purpose |
|-------|---------|
| `0088-Disable-GSC-TrustyVM-EC-HSUM-and-add-RTL8821A` | Combined patch: disables GSC, TrustyVM, EC, HSUM; adds RTL8821A WiFi |
| `0090-Set-BOARD_GRUB_CFG` | Points to grub.cfg.template |
| `0092-Enable-quiet-boot-SOF-firmware` | Quiet boot + Intel-signed audio firmware |

### Firefly Device (new_files/device/google/desktop/firefly/)

Firefly is a WCL (Wildcat Lake) variant that inherits from fatcat. Based on **WW02 BKC**.

| File | Purpose |
|------|---------|
| `AndroidProducts.mk` | Defines firefly lunch targets |
| `BoardConfig.mk` | Inherits fatcat, adds WCL firmware (SOF audio, ISH) |
| `firefly.mk` | Product configuration inheriting fatcat_common |
| `init.firefly.rc` | Firefly-specific init scripts |

### Other Patches

| Patch | Purpose |
|-------|---------|
| `vendor/google/desktop/0001-*` | GRUB/ESP build scripts, boot_part_uuid support |
| `build/make/0001-*` | ESP build rule in core Makefile |
| `frameworks/.../bootanimation/0001-*` | BGRT boot logo support |

## Testing

```bash
# Build ESP image
m esp_image

# Test with QEMU
vendor/google/desktop/layout/test_with_qemu \
    --image=$OUT/android-desktop_image.bin --kvm

# Write to USB
sudo dd if=$OUT/android-desktop_image.bin of=/dev/sdX bs=4M status=progress
```

## Boot Menu Options

The GRUB menu provides:
- **AluminiumOS** - Normal quiet boot
- **AluminiumOS (Verbose)** - Serial console logging enabled
- **AluminiumOS (SELinux Permissive)** - For debugging
- **AluminiumOS (Recovery)** - Boot to recovery mode
- **Advanced Options** - ADB root, fastboot mode, etc.

## Author

TsaiGaggery <gaggery.tsai@intel.com>

## Last Updated

January 2026
