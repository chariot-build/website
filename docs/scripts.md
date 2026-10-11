---
icon: lucide/file-code
---

# Scripts

Scripts are code snippets used to do various things. Currently there are four places where scripts
are used:

- [Source](sources.md) &rarr; `prepare`
- [Package](packages.md) &rarr; `configure`
- Package &rarr; `build`
- Package &rarr; `install`

Scripts are run with `bash -e`, so the first failing command aborts the script and a non-zero exit
fails the build.

## Environment

Scripts run in an isolated container to maximize ease of reproducibility. The root filesystem is
read-only, the designated directories for work are accessible via
[environment variables](#environment-variables).

The container gets fresh user, PID and mount namespaces, and scripts run as an unprivileged user
rather than as root, so they cannot see or touch processes and mounts on the host. They *can* reach
the network, which is what makes it possible to fetch sources, but a build that downloads things
itself will not be reproducible. Declare a [source](sources.md) instead.

### Environment Variables

Chariot exposes various things into the runtime using environment variables.

| Variable      | Availability                    | Description                                                                      |
| ------------- | ------------------------------- | -------------------------------------------------------------------------------- |
| `PARALLELISM` | Everywhere                      | Ideal number of threads to use for parallel builds.                              |
| `SOURCES_DIR` | Everywhere                      | Path to the directory where sources are available.                                |
| `SYSROOT_DIR` | Everywhere                      | Path to the sysroot containing target package dependencies.                       |
| `SOURCE_DIR`  | `prepare`, and any package step when the package declares a `source` | Path to the source directory.         |
| `BUILD_DIR`   | `configure`, `build`, `install` | Path to the build directory.                                                      |
| `INSTALL_DIR` | `install`                       | Path to the install directory.                                                    |
| `PREFIX`      | `configure`, `build`, `install` | Install prefix the package should install itself under.                           |
| `ARCH`        | `configure`, `build`, `install` | Target architecture for packages, host architecture for tools.                    |
| `HOME`        | Everywhere                      | Points at the script's working directory.                                         |

Your own variables come from two places, with the package winning on collisions:

```toml title="chariot_config.toml"
[global_environment_variables]
LC_ALL = "C"
```

```lua title="chariot.lua"
env = { CFLAGS = "-O2 -pipe" }
```

Both are part of the package's content hash, so changing either triggers a rebuild.

### Parallelism

All builds share one job budget, set with `-j`/`--parallelism` and defaulting to the number of
available cores. Chariot hands that budget out through a jobserver, so the total number of jobs
across every package being built at once stays within it.

Build systems that speak the make jobserver protocol pick this up on their own. Everything else
should be told explicitly, and anything that can run in parallel should be:

```bash
make -j"${PARALLELISM}"
ninja -j"${PARALLELISM}"
cmake --build . --parallel "${PARALLELISM}"
```

## Writing good scripts

Keep in mind that the source tree is read-only. Build systems that write into their own source
directory need either an out of tree build:

```bash
"${SOURCE_DIR}"/configure --prefix="${PREFIX}"
```

or a `prepare` script that produces the generated files once, as part of the cached source.

`install` is the only step whose output is kept, and only what lands in `$INSTALL_DIR`. A common
shape is:

```bash
DESTDIR="${INSTALL_DIR}" make install
find "${INSTALL_DIR}/${PREFIX}/bin" -maxdepth 1 -type f -exec strip {} + || true
```

The build directory, by contrast, is wiped before each build unless you explicitly
[enable build directory caching](caching.md#build-directories) for the package.

## Debugging a script

`chariot exec --build-env <package>` assembles exactly the environment that package builds in and
runs a command of your choice in it, which is the fastest way to find out why a `configure`
invocation behaves differently than expected:

```console
$ chariot exec --arch x86_64 --build-env coreutils --stdin bash
```

Adding `-b /build=coreutils` mounts the package's build directory read-only at `/build`, so you can
inspect the output of a failed build. See the [CLI reference](cli.md#chariot-exec).
