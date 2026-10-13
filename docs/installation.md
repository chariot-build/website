---
icon: lucide/download
---

# Installation

## Requirements

Chariot ships as a single static binary with no runtime dependencies. However, chariot leverages Linux unshare calls and overlayfs.
This means Chariot can only run under a Linux kernel with unprivileged user namespaces and overlayfs support.
Luckily the vast majority of Linux distros compile kernels with support for both by default.

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
