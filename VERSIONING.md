# Versioning And Releases

This repository publishes the `satellite1-rpi-setup` Debian source package. A
public release is one immutable binary package build for one target Debian
distribution.

## Debian Package Versions

The version in the first entry of `debian/changelog` is the authoritative
public package version. It uses Debian's standard format:

```text
<feature-version>-<package-revision>
```

Examples:

```text
1.2-1  First packaged release of feature set 1.2
1.2-2  Dependency, packaging, configuration, or compatibility update to 1.2
1.3-1  First packaged release of feature set 1.3
```

Increment the feature version only for a material change to the setup package's
functionality. Increment the package revision for a change such as updating the
required kernel package, installation behavior, or package metadata.

## Distribution Streams

Trixie and Bookworm are separate compatibility and release streams. The binary
package name includes the target distribution, and it must depend on the exact
kernel package built for that distribution.

```text
satellite1-rpi-setup-trixie
satellite1-rpi-setup-bookworm
```

This `develop` branch is the temporary Trixie stream and hardcodes its target
as Trixie. It will later be renamed to `trixie`. A future `bookworm` branch
will hardcode Bookworm instead; builds must not select their distribution at
dispatch time.

For example, a Trixie setup package may depend on:

```text
linux-image-6.18.39-fusb302-trixie-rpi-v8
```

Do not release or bump both distributions merely to synchronize version
numbers. Release only the distribution whose package contents or dependency
contract changed and has been tested.

A shared setup feature may be released for each supported distribution after it
has been validated there. A kernel ABI or kernel-package-name change requires a
new setup package revision only for the affected distribution. A new revision
of an otherwise compatible kernel package does not require a setup release.

Distribution separation is provided by the release tag, the release itself,
and the exact kernel dependency. If packages are later published through an APT
archive, each distribution must have its own suite; packages from different
distributions must not be mixed in one suite.

## Git Tags And GitHub Releases

Public releases are created only from manually created, annotated Git tags.
The tag format is:

```text
v<debian-version>-<distribution>
```

Examples:

```text
v1.2-2-trixie
v1.2-2-bookworm
```

The GitHub Release title is the same tag without the `v` prefix, for example
`1.2-2-trixie`. The release contains the package asset, for example:

```text
satellite1-rpi-setup-trixie_1.2-2_arm64.deb
```

The release tag, Debian package version, target distribution, package
architecture, and exact kernel dependency form the package identity. CI must
verify these values before publishing a release.

Pushes to the distribution stream and manually dispatched builds may upload
GitHub Actions artifacts, but they must not create Git tags or GitHub Releases.

## Local Builds

Developer builds are not public releases. A local build must use a derived
Debian version that sorts below its corresponding public version:

```text
1.2-2~local.<build-id>
```

The `~local` suffix is required because Debian version ordering places it below
`1.2-2`. Local builds therefore cannot consume a public version index or block
a later public upgrade. The local version entry is generated only in ignored
build staging; it must never modify the source `debian/changelog`.

Local builds must be visibly marked as local in their output path and build
logs. They must not create or update a public tag, GitHub Release, or release
asset.

## Release Checklist

1. Update `debian/changelog` with the next public Debian version and a concise
   description of the change.
2. Set and verify the exact kernel dependency for the target distribution.
3. Build from a clean staging directory and validate the resulting package
   metadata, contents, and dependency.
4. Commit the release changes to the target distribution stream.
5. Create an annotated tag in the form `v<debian-version>-<distribution>` on
   that exact commit.
6. Push the tag. CI builds, validates, and publishes the matching GitHub
   Release and `.deb` asset.
