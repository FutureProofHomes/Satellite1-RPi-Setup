# satellite1-rpi-setup — Raspberry Pi Configuration Package

Debian package that configures Raspberry Pi OS for the Satellite1 HAT. Installs device tree overlays, ALSA configuration, and boot-time initialization.

> **⚠️ Early-stage development:**
> This is early-stage experimental software. No official support is provided yet. 
> For issues and feature requests, open an issue on the GitHub repository: https://github.com/futureproofhomes/Satellite1-RPi/issues

## Overview

This package configures the Raspberry Pi to work with the Satellite1 HAT by:

- Enabling required interfaces (SPI, I²S, I²C) in `/boot/firmware/config.txt`
- Installing custom device tree overlays:
  - `satellite1-i2s` — I²S audio support for the HAT
  - `fusb302b` — USB-C Power Delivery controller
- Optionally adding the AHT20 temperature/humidity sensor overlay
- Installing ALSA configuration for Satellite1 audio devices
- Disable legacy analog audio to prevent conflicts
- Loading the `i2c-dev` kernel module at boot
- Creating a systemd service that initializes the DAC and IO expander at boot

## Prerequisites

- Raspberry Pi OS (Trixie) on a Raspberry Pi Zero W2
- Custom kernel with FUSB302 support (`linux-image-6.18.32-fusb302-rpi-v8` or compatible)

## Installation

The package declares a dependency on `linux-image-6.18.32-fusb302-rpi-v8`. Ensure this kernel is installed before or during package installation.

Install the setup package which configures overlays, ALSA, and boot services:

```bash
sudo dpkg -i satellite1-rpi-setup/out/satellite1-rpi-setup_1.0-1_arm64.deb
```

This will:

- Enable SPI, I²S, and I²C interfaces in `/boot/firmware/config.txt`
- Install custom device tree overlays (`satellite1-i2s`, `fusb302b`)
- Copy ALSA configuration to `/etc/alsa/conf.d/50-satellite1.conf`
- Load the `i2c-dev` kernel module at boot
- Disable legacy analog audio to prevent conflicts

Reboot to apply kernel configuration changes:

```bash
sudo reboot
```

### Automatic kernel dependency

The package declares a dependency on `linux-image-6.18.32-fusb302-rpi-v8`. Ensure this kernel is installed before or during package installation.

## Build Process

### Prerequisites

- Docker
- `make`
- Git

### Source layout

```
satellite1-rpi-setup/
├── debian/           # Debian packaging metadata
│   ├── control.in    # Package dependencies template
│   ├── install.in    # File installation manifest
│   ├── postinst.in   # Post-install script template
│   └── rules         # debhelper build rules (compiles .dts → .dtbo)
├── dt-overlays/      # Device tree overlay sources (.dts)
│   ├── satellite1-i2s.dts
│   └── fusb302b.dts
├── etc/              # Configuration files installed to system
│   └── alsa/conf.d/50-satellite1.conf
├── docker/           # Docker build environment
│   └── Dockerfile.deb.trixie.arm64
└── Makefile          # Build orchestration
```

### Build steps

```bash
# Build the shared Docker image (once)
make image

# Build the .deb package
make deb
```

Built packages are placed in `out/`.

The build process:

1. Compiles device tree overlays (`.dts` → `.dtbo`) using `dtc`
2. Generates `DEBIAN/control` from template with package name/version/kernel
3. Generates `DEBIAN/postinst` with kernel release substitution
4. Packages everything into a `.deb` using `dpkg-buildpackage`

### Build variables

Override these on the command line:


| Variable            | Default                  | Description           |
| --------------------- | -------------------------- | ----------------------- |
| `PACKAGE_NAME`      | `satellite1-rpi-setup`   | Package name          |
| `ACTIVATOR_VERSION` | `1.0`                    | Package version       |
| `ARCH`              | `arm64`                  | Target architecture   |
| `KERNEL_RELEASE`    | `6.18.32-fusb302-rpi-v8` | Kernel version string |
| `OUT_DIR`           | `$(PWD)/out`             | Output directory      |

Example:

```bash
make deb KERNEL_RELEASE=6.18.32-fusb302-rpi-v8 ACTIVATOR_VERSION=1.0
```

### Local build (without Docker)

While possible, building directly on the host is not recommended. Use the Docker-based workflow for reproducibility.

## What gets installed

| File/Directory                       | Destination                        | Purpose                                    |
| -------------------------------------- | ------------------------------------ | -------------------------------------------- |
| `*.dtbo` (compiled overlays)         | `/boot/firmware/overlays/`         | Device tree overlays loaded by kernel      |
| `etc/alsa/conf.d/50-satellite1.conf` | `/etc/alsa/conf.d/`                | ALSA PCM device configuration              |
| `postinst` script                    | `DEBIAN/` (run at install)         | Configures`config.txt` and sets up modules |
| `target-kernel` marker               | `/usr/share/satellite1-rpi-setup/` | Records expected kernel version            |

## Post-install behavior

The `postinst` script (executed automatically on `dpkg -i`) performs these actions on every install/upgrade:

1. Copies `.dtbo` files to the boot overlays directory
2. Backs up `/boot/firmware/config.txt` (once, to `config.txt.satellite1.bak`)
3. Enables these `dtparam` directives:
   - `dtparam=spi=on`
   - `dtparam=i2s=on`
   - `dtparam=i2c_arm=on`
   - `dtparam=i2c_arm_baudrate=100000`
4. Adds `dtoverlay` lines:
   - `dtoverlay=satellite1-i2s`
   - `dtoverlay=fusb302b`
   - `dtoverlay=i2c-sensor,addr=0x38,chip=aht20`
5. Enables `i2c-dev` module via `/etc/modules-load.d/i2c.conf`
6. Comments out `dtparam=audio=on` to avoid conflicts with Satellite1 audio

## Verification

After install and reboot:

```bash
# Check kernel version
uname -r
# Should show: 6.18.32-fusb302-rpi-v8

# Verify overlays are present
ls /boot/firmware/overlays/ | grep -E 'satellite1|fusb302'
# Should list: satellite1-i2s.dtbo, fusb302b.dtbo

# Check config.txt entries
grep -E 'dtparam|dtoverlay' /boot/firmware/config.txt

# Verify i2c-dev is loaded
lsmod | grep i2c_dev
```

## Uninstall

```bash
sudo dpkg --remove satellite1-rpi-setup
```

Note: The package does **not** remove modifications made to `config.txt` to preserve user changes. Backups are kept at `config.txt.satellite1.bak`.

## Dependencies

Depends on:

- `linux-image-6.18.32-fusb302-rpi-v8` (custom kernel)
- `flashrom` (for firmware flashing utilities)
- `libasound2-plugins` (ALSA plugin support)
- `python3-venv`, `python3-pip` (for SDK installation)

Recommends:

- `alsa-utils` (command-line audio tools)
- `lm-sensors` (hardware monitoring)

## Related

- Builds along with `satellite1-rpi-sdk` and `rpi-kernel-fusb302` via the top-level `satellite1-rpi` project Makefile
- Part of the complete Satellite1 SDK distribution

## License

See the top-level LICENSE file.

## Repository

https://github.com/futureproofhomes/Satellite1-RPi/tree/main/satellite1-rpi-setup
