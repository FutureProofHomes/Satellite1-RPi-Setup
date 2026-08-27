# Satellite1 Raspberry Pi Setup

Debian packaging for the Satellite1 Raspberry Pi configuration. The package
installs device-tree overlays, ALSA configuration, and boot-time setup for the
Satellite1 HAT.

> This is early-stage experimental software. No official support is provided.

## Package

This repository builds `satellite1-rpi-setup-trixie` for Raspberry Pi OS
(Trixie).

## Build A Local Package

Prerequisites:

- Docker
- `make`
- Git

Build a local package with:

```sh
make deb
```

The build uses Docker and writes its artifact to `out/local/`. Local packages
are clearly marked and sort below their corresponding public package version:

```text
satellite1-rpi-setup-trixie_<public-version>~local.<build-id>_arm64.deb
```

Inspect the active build configuration with:

```sh
make print-config
```

The package build compiles device-tree overlays, generates the Debian package
metadata and post-install script, and builds the `.deb` with
`dpkg-buildpackage`.

## Versioning

The first entry in `debian/changelog` is the authoritative public package
version. Debian versions use this format:

```text
<feature-version>-<package-revision>
```

Examples:

- `1.2-1`: first packaged release of feature set 1.2
- `1.2-2`: dependency, packaging, configuration, or compatibility update
- `1.3-1`: first packaged release of feature set 1.3

Increment the feature version for a material setup-package feature change.
Increment the package revision for changes such as a kernel meta-package dependency,
installation behavior, or package metadata.

Local builds derive a version such as `1.2-2~local.<build-id>` in ignored build
staging only. They never modify `debian/changelog` or consume a public package
version.

## Releases

CI builds local artifacts for development and manual verification. To prepare a
public release, manually run the **Prepare Trixie release** workflow from the
current `develop` tip. The workflow:

- Reads the public version from `debian/changelog`.
- Builds and validates the corresponding public package.
- Creates the annotated `v<debian-version>-trixie` tag, for example
  `v1.2-2-trixie`.
- Creates a GitHub Release as a draft with the validated `.deb` attached.
- Generates the draft release body from the top `debian/changelog` entry.

Review and edit the draft release before publishing it.

## Public Release Checklist

1. Update `debian/changelog` with the next public version and release notes.
2. Set and verify the Raspberry Pi OS (Trixie) FUSB302 kernel meta-package dependency.
3. Build and inspect the package from a clean tree.
4. Commit and merge the release changes to `develop`.
5. Run the **Prepare Trixie release** workflow from the current `develop` tip.
6. Review the generated draft GitHub Release and publish it.

## Installation

Download the setup package plus the matching kernel image and meta-package
assets from their published GitHub Releases, then install them together:

```sh
sudo apt install \
  ./linux-image-<kernel-release>_<revision>_arm64.deb \
  ./linux-image-fusb302-trixie-rpi-v8_<kernel-version>-<revision>_arm64.deb \
  ./satellite1-rpi-setup-trixie_<version>_arm64.deb
```

The setup package depends on the stable FUSB302 Trixie kernel meta package,
which in turn depends on its matching kernel image. Reboot after installation
to apply boot configuration changes.

Installing this package with APT replaces the legacy
`satellite1-rpi-setup` package automatically.

## Installed Files And Behavior

The package:

- Installs `satellite1-i2s` and `fusb302b` device-tree overlays, then copies
  them to the firmware overlays directory during installation.
- Installs ALSA configuration at `/etc/alsa/conf.d/50-satellite1.conf`.
- Stores package assets below `/usr/share/satellite1-rpi-setup-trixie/`.
- Configures required SPI, I2S, and I2C boot settings.
- Loads the `i2c-dev` kernel module and disables legacy analog audio to avoid
  card conflicts.

The post-install script preserves existing boot configuration changes and
creates a one-time `config.txt.satellite1.bak` backup.

## Verification

After installation and reboot:

```sh
uname -r
ls /boot/firmware/overlays/ | grep -E 'satellite1|fusb302'
grep -E 'dtparam|dtoverlay' /boot/firmware/config.txt
lsmod | grep i2c_dev
```

## Uninstall

```sh
sudo apt remove satellite1-rpi-setup-trixie
```

Uninstalling does not remove boot configuration changes, to preserve user
changes. The backup remains at `config.txt.satellite1.bak`.

## License

See the top-level LICENSE file.
