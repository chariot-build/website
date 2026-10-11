---
icon: lucide/box
---

# Packages

A package is one buildable unit. Chariot builds it in an isolated container, captures what the
`install` script writes, and turns that into a binary package other builds can depend on.

## Tools vs packages

Every declaration says where the software it produces is meant to run:

```lua
Tool { name = "nasm", ... }
Package { name = "coreutils", ... }
```

- A **`Tool`** produces software that runs on your machine, so that other builds can use it. A cross
  compiler, an assembler or a code generator are tools.
- A **`Package`** produces software that runs on the system you are building, and ends up in it.

Both are built the same way, in the same kind of container on your machine. The difference is who
the result is for, which is why a cross compiler is a `Tool` while the C library it targets is a
`Package`. Two things follow from that distinction:

- **How they are built.** A tool is built for the architecture Chariot itself runs on. A package is
  built for the `--arch` architecture, using `target_prefix` as its install prefix. Scripts see both
  as [`ARCH` and `PREFIX`](scripts.md#environment-variables).
- **What depending on them means.** A tool you depend on becomes available to run during your build.
  A target package you depend on is installed into the sysroot, so you can compile and link against
  it.

Names are unique per flavour, so a host `gcc` and a target `gcc` can coexist. On the command line,
tools are selected with `--tool`.

## Fields

```lua
local coreutils = Package {
    name = "coreutils",
    version = "9.5",
    revision = 1,
    source = coreutils_source,
    dependencies = {
        "build-essential",
        gcc_tool,
        mlibc,
    },
    runtime_dependencies = { mlibc },
    env = {
        CFLAGS = "-O2 -pipe",
    },
    configure = [[
        "${SOURCE_DIR}"/configure \
            --host="${ARCH}-chariot" \
            --prefix="${PREFIX}" \
            --with-sysroot="${SYSROOT_DIR}"
    ]],
    build = [[
        make -j"${PARALLELISM}"
    ]],
    install = [[
        DESTDIR="${INSTALL_DIR}" make install
    ]],
}
```

| Field                  | Type                              | Required | Description                                                                            |
| ---------------------- | --------------------------------- | -------- | -------------------------------------------------------------------------------------- |
| `name`                 | string                            | yes      | Package name. Must not contain whitespace.                                             |
| `version`              | string                            | yes      | Upstream version. Must not contain whitespace, `-` or `_`.                             |
| `revision`             | number                            | yes      | Packaging revision, starting at `1`. See below.                                        |
| `source`               | source reference                  | no       | The source to build, available to scripts as `SOURCE_DIR`.                             |
| `dependencies`         | array                             | no       | Everything needed at build time. See [Dependencies](#dependencies).                    |
| `runtime_dependencies` | array of package references       | no       | Packages required to run this one. See [Runtime dependencies](#runtime-dependencies).  |
| `env`                  | map of string to string or number | no       | Extra environment variables for all of this package's scripts.                         |
| `configure`            | string                            | no       | First [script](scripts.md).                                                            |
| `build`                | string                            | no       | Second [script](scripts.md).                                                           |
| `install`              | string                            | yes      | Final [script](scripts.md). Everything it writes to `INSTALL_DIR` becomes the package. |

Chariot does not use `revision` for anything itself. Rebuilds are decided by content hashing, so a
revision bump is not how you force one. It exists because it ends up in the produced package's
metadata, where the target's package manager uses it to order upgrades. That makes it worth keeping
accurate for target packages, and largely irrelevant for host tools.

`Package` and `Tool` return a reference that can be used in other declarations' `dependencies` and
`runtime_dependencies`. That is the only way to express a relationship between two packages, which
is why configs usually assign them to locals:

```lua
local mlibc = Package { name = "mlibc", ... }
local coreutils = Package { name = "coreutils", dependencies = { mlibc }, ... }
```

## Dependencies

The `dependencies` array mixes four kinds of entry, distinguished by what you put in it:

| Entry                 | Kind           | Effect                                                         |
| --------------------- | -------------- | -------------------------------------------------------------- |
| `"build-essential"`   | Native package | Installed into the container from the rootfs distribution.     |
| a `Tool` reference    | Host tool      | Built first, then made available to run during the build.      |
| a `Package` reference | Target package | Built first, then installed into the sysroot at `SYSROOT_DIR`. |
| a `Source` reference  | Source         | Fetched and made available read-only under `SOURCES_DIR`.      |

```lua
dependencies = {
    "build-essential", "texinfo",   -- from the rootfs
    binutils_tool, gcc_tool,        -- host tools, available to run
    mlibc,                          -- target package, lands in the sysroot
    gnulib_source,                  -- extra source tree
}
```

Native packages come from the [rootfs](rootfs.md) distribution and are the one dependency kind
Chariot does not build. Declaring them per package, rather than installing them into the rootfs
once, means each build only sees the native tooling it actually asked for. Packages that every
build needs belong in `global_native_packages` instead.

### Runtime dependencies

`runtime_dependencies` are written into the produced package's metadata as
`<name> >= <version>_<revision>` so the target's package manager can resolve them at install time.
They must be the same flavour as the package declaring them, so a `Tool` cannot have a target
`Package` as a runtime dependency.

They are not build inputs, but they do travel with the packages that declare them: when a build
depends on package A, A's runtime dependencies are installed into the sysroot alongside it. So if A
has a runtime dependency on B, a package depending on A gets both.

A library you both link against and ship a dependency on belongs in both lists:

```lua
dependencies = { mlibc },
runtime_dependencies = { mlibc },
```

## Build lifecycle

Building a package happens in five stages:

1. **Native package set.** The project's `global_native_packages` plus this package's native
   dependencies are assembled into the environment.
2. **Dependencies.** Host tools and target packages are built if needed, then installed: tools so
   they can run, packages into a fresh sysroot. Sources are made available read-only.
3. **Build directory.** A directory for the build is prepared and exposed as `BUILD_DIR`. It starts
   empty unless [build directory caching](caching.md#build-directories) is enabled for the package.
4. **Scripts.** `configure`, `build` and `install` run in that order, each starting in `BUILD_DIR`.
   A non-zero exit stops the build and prints the captured log.
5. **Packaging.** Everything the `install` script wrote to `INSTALL_DIR` is packaged, with the name,
   version, revision, architecture and runtime dependencies attached as metadata.

Stages 1 to 4 are keyed on the package's content hash and stage 5 on its metadata hash, so changing
only `version` or `revision` repackages without rebuilding. See [Caching](caching.md).

## Building and installing

```console
$ chariot build --arch x86_64 coreutils          # target package
$ chariot build --arch x86_64 --tool nasm        # host tool
$ chariot install --arch x86_64 coreutils /mnt   # build and install into /mnt
$ chariot lookup --arch x86_64 coreutils         # where is the built package?
```

`build` stops after producing the package. `install` additionally unpacks it into a directory of
your choice, which is how a full system image gets assembled. Multiple packages can be given at
once, and `--keep-going` keeps building unrelated packages after a failure instead of stopping at
the first one.

To debug a build, `chariot exec --build-env <package>` drops you into the exact environment that
package builds in. See the [CLI reference](cli.md#chariot-exec).
