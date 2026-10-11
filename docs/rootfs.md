---
icon: lucide/hard-drive
---

# Root Filesystem

Every build runs inside a container whose root filesystem is managed by Chariot. The rootfs is what
makes builds reproducible between machines: the compiler, shell and utilities a build sees come from
a pinned filesystem rather than from whatever happens to be installed on your host.

## Manifests

A rootfs is described by a **manifest**, a TOML document that says which base archives to unpack,
which directories and files must exist, how to download and install native packages, and which
package provides which binary.

Your project pins one manifest by version and hash:

```toml title="chariot_config.toml"
[rootfs]
version = "debian/20260901T000000Z"
hash = "7a9147a2fce8ac2bae29fcde0b822ce613ffc73b66acf080c70301f386afee81"
```

The manifest is fetched from a URL template and verified against `hash` with SHA-256 before anything
is used. The default template is:

```text
https://cdn.chariot-build.dev/manifests/@ARCH@/@VERSION@.toml
```

`@ARCH@` is replaced with the architecture Chariot is running on and `@VERSION@` with the version
above. Override it with `rootfs.url` to serve manifests yourself; a custom URL must keep both
placeholders.

Because the hash covers the manifest, and the manifest covers the hashes of the archives it unpacks,
pinning one hash pins the entire base environment. The manifest hash is also mixed into every
package's content hash, so changing the rootfs rebuilds the project.

## Initialization

On the first build, Chariot downloads and sets up the rootfs in `.chariot-rootfs`
(`--rootfs`, `CHARIOT_ROOTFS_PATH`). The directory gitignores itself. You can also set it up
explicitly:

```console
$ chariot rootfs init debian/20260901T000000Z 7a9147a2fce8ac2bae29fcde0b822ce613ffc73b66acf080c70301f386afee81
```

To check what an existing rootfs is:

```console
$ chariot rootfs status
```

If the rootfs on disk does not match what the config asks for, Chariot refuses to build:

- **Hash mismatch.** The rootfs is for a different manifest. Delete the rootfs directory and let it
  be set up again.
- **Version mismatch but matching hash.** A warning only. The manifest content is right but the
  recorded version is not what the config says, which usually means a manifest was republished.

## Native package sets

Native packages are the packages that come from the rootfs distribution itself, referenced as plain
strings in a [package's dependencies](packages.md#dependencies) or in `global_native_packages`.
Chariot does not build them; it downloads and installs them using the commands the manifest
provides.

Rather than installing them into the rootfs, Chariot builds **package sets**: cached, read-only
directories that are stacked onto the rootfs with overlayfs when a build runs. A set can be based on
another set, so the common case of "the global packages, plus two extra for this one package" reuses
the global set instead of installing everything again. Files shared between sets are hardlinked.

The point of all this is control: each package declares the native tooling its build needs, and gets
exactly that, without every other build in the project inheriting it.

```console
$ chariot rootfs pkgset list
```

```text
id     state        base   depth  size         packages
------------------------------------------------------------
1      deduped             0      412.3MiB     build-essential
2      cached       1      1      18.7MiB      texinfo, help2man
```

`state` is `cached` for a freshly created set and `deduped` once its files have been hardlinked
against the others. `base` and `depth` show the stacking.

To warm a set up front, for example in a CI image:

```console
$ chariot rootfs pkgset cache build-essential texinfo
```

To drop every set that is not currently in use:

```console
$ chariot rootfs purge
```

Sets are rebuilt on demand, so purging costs download time, not correctness.

## Manifest format

Writing a manifest is an advanced task, usually only needed to support a distribution Chariot does
not publish manifests for. A manifest looks like this:

```toml
manifest_version = 6

directories = ["/run", "/tmp", "/proc", "/dev/pts", "/dev/shm", "/chariot"]
files = ["/etc/resolv.conf", "/dev/null", "/dev/zero"]

[[archives]]
url = "https://example.com/debian-base-20260901.tar.xz"
hash = "0c7a1d0b2f8e4a6c9b3d5f7081a2c4e6183a5c7e9b0d2f416385a7c9e1b3d5f7"
compression = "xz"
subdir = "rootfs"

[commands]
setup = "apt-get update && apt-get install -y @ROOT_PACKAGES@"
pkg_download = "cd /var/cache/chariot && apt-get download @PACKAGE@"
pkg_install = "dpkg --install /var/cache/chariot/@PACKAGE@*.deb"

[packages]
root = ["bash", "python3", "xbps"]

[packages.binary_map]
wget = "wget"
sha256sum = "coreutils"
bsdtar = "libarchive-tools"
unzip = "unzip"
git = "git"
patch = "patch"
clangd = "clangd"
```

| Key                     | Description                                                                                                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `manifest_version`      | The manifest format version. A manifest written for a version your Chariot does not expect is rejected rather than misinterpreted.                                                                      |
| `directories`           | Directories created in the rootfs before setup runs.                                                                                                                                                    |
| `files`                 | Empty files created in the rootfs, along with their parent directories.                                                                                                                                 |
| `archives`              | Base archives unpacked into the rootfs, in order. See below.                                                                                                                                            |
| `commands.setup`        | Shell command run once as root inside the fresh rootfs. `@ROOT_PACKAGES@` is replaced with `packages.root`, space separated.                                                                            |
| `commands.pkg_download` | Shell command that downloads one native package. `@PACKAGE@` is replaced with its name. Runs as root in the rootfs, so downloads can be kept there.                                                     |
| `commands.pkg_install`  | Shell command that installs one downloaded native package. `@PACKAGE@` is replaced with its name. Runs as root with the new package set as the writable layer, so everything it writes becomes the set. |
| `packages.root`         | The base packages installed into the rootfs during setup. Available to every build.                                                                                                                     |
| `packages.binary_map`   | Map from binary name to the package that provides it. See below.                                                                                                                                        |

Each entry in `archives` takes:

| Key           | Required | Description                                                                              |
| ------------- | -------- | ---------------------------------------------------------------------------------------- |
| `url`         | yes      | Where to download the tar archive from.                                                  |
| `hash`        | yes      | SHA-256 of the downloaded file, verified before unpacking.                               |
| `compression` | yes      | `xz` or `zstd`.                                                                          |
| `subdir`      | no       | Unpack only this subdirectory of the archive into the rootfs instead of the whole thing. |

### Binaries Chariot needs

Chariot uses the binary map to install the tooling it needs for its own work, on demand, without
putting it in every build environment. A manifest that cannot provide one of these fails immediately
with `This rootfs manifest is missing a required package mapping for the ... binary`:

| Binary      | Used for                      |
| ----------- | ----------------------------- |
| `wget`      | Downloading archives          |
| `sha256sum` | Verifying archive checksums   |
| `bsdtar`    | Extracting tar archives       |
| `unzip`     | Extracting zip archives       |
| `git`       | Fetching git sources          |
| `patch`     | Applying patches              |
| `clangd`    | Only for `chariot lsp clangd` |

### What the rootfs itself must contain

The filesystem the manifest produces has to satisfy a few requirements, because Chariot's container
runtime and packaging rely on them.

It must contain:

- `bash`, `python3` and `xbps`
- A `root` user with uid and gid `0`, and a `chariot` user with uid and gid `1000`

It must contain these paths, which the container runtime mounts over or into:

| Kind        | Paths                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Files       | `/dev/tty`, `/dev/null`, `/dev/zero`, `/dev/full`, `/dev/random`, `/dev/urandom`, `/etc/resolv.conf` |
| Directories | `/dev/pts`, `/dev/shm`, `/run`, `/tmp`, `/proc`, `/chariot`                                          |
