---
icon: lucide/hammer
---

# Introduction

Chariot is a modern meta build system written in Rust used for building operating system distributions.

!!! warning

    Chariot is currently undergoing major changes and is preparing for version 3.
    Chariot and the documentation may not be complete, stable or working at this moment.

## Why Chariot

Building a distribution means building hundreds of small, interdependent things: host tools that run
on your machine, target packages that run on the system you are creating, and the sources both are
built from. Chariot describes all of that in one [Lua](configuration.md) config, builds it in
isolated containers, and caches every step by content hash so that nothing is ever built twice.

- Zero runtime dependencies, shipped as a static binary
- Parallel package building with a shared job budget
- Flexible Lua based config format
- Custom rootfs support
- Optimized for disk space using hardlinks and overlayfs
- Hash based package invalidation

## Concepts

| Concept                                           | What it is                                                                                                   |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [Source](sources.md)                              | Something to build from: an archive, a git revision or a local directory, optionally patched and prepared.    |
| [Package](packages.md)                            | Software built to run on the system you are building. Declared with `Package`.                                |
| [Tool](packages.md#tools-vs-packages)             | Software built to run on your machine so that other builds can use it, such as a compiler or `nasm`. Declared with `Tool`. |
| [Native package](packages.md#dependencies)        | A package that comes from the rootfs distribution itself, referenced by name as a plain string.               |
| [Script](scripts.md)                              | The `prepare`, `configure`, `build` and `install` snippets that do the actual work inside the container.       |
| [Rootfs](rootfs.md)                               | The base filesystem every build runs in, described by a hash pinned manifest and managed by Chariot.          |
| [Store and ledger](caching.md)                    | Where build results live and how Chariot knows a result is still valid.                                       |
| [Profile](configuration.md#options-and-profiles)  | One permutation of target architecture and user defined options.                                              |

## CLI Quick Reference

Every build related command needs `--arch` (or `CHARIOT_ARCH`) to know what target architecture to
build for. See the [CLI reference](cli.md) for all commands and flags.

| Command                                                      | What it does                                                            |
| ------------------------------------------------------------ | ----------------------------------------------------------------------- |
| `chariot build --arch <arch> [--tool] <packages>...`          | Build the given packages                                                |
| `chariot install --arch <arch> [--tool] <packages>... <dest>` | Build the given packages and install them into `<dest>`                 |
| `chariot lookup --arch <arch> [--tool] <name>`                | Print the store path of an already built package, or exit `1`           |
| `chariot exec --arch <arch> [options] <script>`               | Run a command inside a build environment                                |
| `chariot lsp --arch <arch> clangd <package>`                  | Run a language server inside a package's build environment              |

## Quick Start Guide

Install Chariot as described in [Installation](installation.md), then create a project directory
with two files in it.

### 1. Declare the rootfs

`chariot_config.toml` is the entry point of a Chariot project. Its directory is the *project root*
that all relative paths are resolved against. The only required section is the rootfs that builds
run in:

```toml title="chariot_config.toml"
[rootfs]
version = "debian/20260901T000000Z"
hash = "7a9147a2fce8ac2bae29fcde0b822ce613ffc73b66acf080c70301f386afee81"
```

The `version` names a manifest and the `hash` pins it, so the base filesystem your builds run in is
always exactly the one you expect. All other keys are optional and documented in
[Configuration](configuration.md).

### 2. Declare something to build

`chariot.lua` is the Lua script Chariot evaluates to learn about your sources, tools and packages.
This one builds a single host tool:

```lua title="chariot.lua"
local NASM_VERSION <const> = "3.02"

Tool {
    name = "nasm",
    version = NASM_VERSION,
    revision = 1,
    source = Source {
        name = "nasm",
        Archive(
            "https://www.nasm.us/pub/nasm/releasebuilds/" .. NASM_VERSION .. "/nasm-" .. NASM_VERSION .. ".tar.xz",
            "87336eba53b4acfe917424ab5d500d2b0054d9f5148d35c2273ccf2cfb712f0d"
        )
    },
    dependencies = {
        "build-essential",
    },
    configure = [[
        LDFLAGS="-static" "${SOURCE_DIR}"/configure --prefix="${PREFIX}"
    ]],
    build = [[
        make -j"${PARALLELISM}"
    ]],
    install = [[
        DESTDIR="${INSTALL_DIR}" make install
    ]]
}
```

`Tool` declares a host package, `Source` declares where to get the code from, and the three scripts
are run in that order inside the container. `Tool`, `Package` and `Source` all return a reference
that can be used in another declaration's `dependencies`, which is how the build graph is built.

### 3. Build it

```console
$ chariot build --arch x86_64 --tool nasm
```

The first run does the setup work: it downloads and verifies the rootfs into `.chariot-rootfs`,
creates the cache in `.chariot-cache`, asks you to confirm the new profile, fetches the source,
and then runs `configure`, `build` and `install`.

Run the same command again and it finishes immediately: every step was recorded by content hash,
so there is nothing left to do.

From here, other packages can depend on `nasm` and use it while they build. Once you have target
packages to assemble into a system, `chariot install` builds them and unpacks them into a directory
of your choice:

```console
$ chariot install --arch x86_64 coreutils bash /mnt/rootfs
```

### Next steps

- [Configuration](configuration.md) for the config files, options and profiles
- [Sources](sources.md) and [Packages](packages.md) for the full Lua API
- [Scripts](scripts.md) for what the build environment looks like
- [Editor Setup](editor-setup.md) to get completion and type checking for `chariot.lua`
