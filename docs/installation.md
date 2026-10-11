---
icon: lucide/download
---

# Installation

## Requirements

Chariot ships as a single static binary with no runtime dependencies, but the builds it runs need a
few things from the host:

- **Linux.** Builds run in a container that Chariot sets up itself using user, PID and mount
  namespaces, so unprivileged user namespaces must be available. Chariot does not need to run as
  root.
- **`overlayfs` usable inside a user namespace.** Chariot stacks rootfs package sets, sources and
  host tools with overlay mounts.
- **One filesystem.** The rootfs, cache, store, work directories and ledger must all live on the
  same filesystem. Chariot moves and hardlinks files between them, which cannot cross a filesystem
  boundary. Both default to the project directory, so this only matters if you point `--cache` or
  `--rootfs` somewhere else.
- **Network access.** The rootfs, its packages and your sources are fetched over the network.
  Build scripts also have network access.
- **Disk space.** A distribution build keeps every intermediate result, so plan for tens of
  gigabytes on a real project.

## Installing Chariot

=== "Prebuilt binary"

    Download a statically linked musl binary from the
    [releases tab](https://github.com/chariot-build/chariot/releases), then make it executable and
    put it somewhere on your `PATH`.

=== "Nix"

    ```console
    $ nix shell github:chariot-build/chariot
    ```

=== "From source"

    Requires a Rust toolchain:

    ```console
    $ cargo install --git https://github.com/chariot-build/chariot
    ```

Check that it works:

```console
$ chariot --version
```

## Shell completions

Chariot can generate completions for `bash`, `zsh`, `fish`, `elvish` and `powershell`:

```console
$ chariot support completions bash
$ chariot support completions zsh
$ chariot support completions fish
```

## Licenses

Chariot is licensed under BSD-3-Clause. To print its license together with the third party notices
of every crate it links against:

```console
$ chariot support licenses
```
