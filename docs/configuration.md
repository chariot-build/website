---
icon: lucide/settings
---

# Configuration

A Chariot project has three files of interest:

| File                  | Purpose                                                                       | Committed? |
| --------------------- | ----------------------------------------------------------------------------- | ---------- |
| `chariot_config.toml` | Global project settings: the rootfs, the target prefix, global environment.    | Yes        |
| `chariot.lua`         | The sources, tools and packages the project builds.                           | Yes        |
| `.chariot.toml`       | Per developer settings: source overrides, build directory caching. Optional.  | No         |

A minimal project looks like this:

```text
my-distro/
├── chariot_config.toml
├── chariot.lua
├── .chariot.toml          # optional, personal
├── .chariot-cache/        # created by chariot
└── .chariot-rootfs/       # created by chariot
```

Both directories Chariot creates gitignore themselves, so they are safe to leave inside a
repository.

## Project root

The directory containing `chariot_config.toml` is the **project root**. Everything resolved from the
config is resolved against it: the Lua entry file, `require` paths, patch files and the paths passed
to `chariot.read_file` and `chariot.list_dir`.

Those paths can never escape the project root. Leading `/` and `./` components are ignored when
joining, so `/etc/passwd` is read as `<project root>/etc/passwd`.

Chariot does not care what your working directory is as long as it can find the base config, so
`--base-config ../my-distro/chariot_config.toml` works from anywhere.

## `chariot_config.toml`

The base config. Only `[rootfs]` is required.

```toml title="chariot_config.toml"
lua_root = "chariot.lua"
target_prefix = "/usr"
global_native_packages = ["build-essential"]

[global_environment_variables]
LC_ALL = "C"

[rootfs]
version = "debian/20260901T000000Z"
hash = "7a9147a2fce8ac2bae29fcde0b822ce613ffc73b66acf080c70301f386afee81"
```

| Key                            | Type                     | Default         | Description                                                                                                 |
| ------------------------------ | ------------------------ | --------------- | ----------------------------------------------------------------------------------------------------------- |
| `lua_root`                     | string, a path           | `chariot.lua`   | The Lua file Chariot evaluates first, relative to the project root.                                         |
| `target_prefix`                | string                   | `/usr`          | Install prefix for target packages, exposed to their scripts as `PREFIX`.                                   |
| `global_native_packages`       | array of strings         | empty           | Native rootfs packages installed into every build environment, on top of each package's own dependencies.  |
| `global_environment_variables` | map of string to string  | empty           | Environment variables exported into every script.                                                           |
| `rootfs.version`               | string                   | required        | Manifest version to use, for example `debian/20260901T000000Z`.                                             |
| `rootfs.hash`                  | string                   | required        | SHA-256 of the manifest. Chariot refuses to use a manifest that does not match.                             |
| `rootfs.url`                   | string                   | CDN default     | Manifest URL template, must contain `@ARCH@` and `@VERSION@`. See [Root Filesystem](rootfs.md).             |

!!! warning

    `target_prefix`, `global_native_packages` and `global_environment_variables` are part of every
    package's content hash. Changing any of them invalidates the whole project and rebuilds
    everything.

## `chariot.lua`

The Lua entry file declares what the project builds. The full API is covered in
[Sources](sources.md) and [Packages](packages.md); this section covers the environment the script
itself runs in.

### Splitting the config up

`require` is available and resolves module paths against the project root, so a config can be split
across as many files and directories as you like:

```lua title="chariot.lua"
require("tools.binutils")
require("tools.gcc")
require("packages.coreutils")
```

### Available Lua standard library

The config runs in a restricted Lua interpreter. `math`, `string`, `table`, `package` and the basic
functions are available. `io`, `os`, `debug`, `coroutine` and `utf8` are **not**. A config cannot
read the outside world except through the `chariot` API below, which keeps config evaluation
reproducible.

### The `chariot` table

| Member                              | Description                                                                                               |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `chariot.target_arch`               | The architecture passed via `--arch`.                                                                     |
| `chariot.target_prefix`             | The effective `target_prefix`.                                                                            |
| `chariot.options`                   | Table of user options passed via `-o KEY=VALUE`, as strings.                                              |
| `chariot.read_file(path)`           | Read a file from the project root and return its contents as a string.                                    |
| `chariot.list_dir(path)`            | List a directory in the project root, returning `{ filename = string, is_dir = boolean }` entries.         |
| `chariot.concat_paths(a, b)`        | Join two path strings.                                                                                    |
| `chariot.def_source(...)`           | Low level source registration. Prefer [`Source`](sources.md).                                             |
| `chariot.def_package(...)`          | Low level package registration. Prefer [`Package` and `Tool`](packages.md).                               |

### Convenience helpers

Chariot adds a few helpers on top of stock Lua:

| Helper                                | Description                                                                 |
| ------------------------------------- | --------------------------------------------------------------------------- |
| `Path(a, b, ...)`                     | Join any number of path segments.                                           |
| `string.starts_with(s, prefix)`       | Returns `true` if `s` starts with `prefix`.                                 |
| `string.ends_with(s, suffix)`         | Returns `true` if `s` ends with `suffix`.                                   |
| `string.split(s, separator)`          | Split into an array of strings. `separator` defaults to whitespace.         |
| `table.merge(t1, t2)`                 | Shallow merge, `t2` wins on collisions.                                     |
| `table.print(t)`                      | Print `key=value` for each entry, useful while debugging a config.          |

Example of picking up every package directory automatically:

```lua
for _, entry in ipairs(chariot.list_dir("packages")) do
    if entry.is_dir then
        require("packages." .. entry.filename)
    end
end
```

## Options and profiles

Options are string key/value pairs you pass on the command line and read from the config:

```console
$ chariot build --arch x86_64 -o debug=true my-package
```

```lua
if chariot.options["debug"] == "true" then
    -- ...
end
```

A **profile** is one permutation of target architecture and options. Because options can change what
the config declares, each profile is a different build. Chariot records the profiles it has seen so
that `chariot cache gc` knows which results are still reachable.

The first time it encounters a new profile it asks for confirmation:

```text
Detected a new profile (profile describes a specific permutation of architecture and options). Proceed?
```

This exists to catch mistakes like `-o debgu=true`, which would otherwise silently build a whole
second distribution. In CI, pass `--allow-new-profiles` to accept without prompting.

Options are not automatically visible to build scripts. To use one inside a script, pass it through
the package's `env`:

```lua
Package {
    name = "my-package",
    env = { DEBUG = chariot.options["debug"] or "false" },
    -- ...
}
```

## `.chariot.toml`

The local config holds settings that belong to a developer rather than to the project. It is
optional, read from `.chariot.toml` in the working directory by default, and should not be
committed.

```toml title=".chariot.toml"
disable_pruning = false

[pkgs.my-kernel]
enable_build_cache = true

[tools.gcc]
enable_build_cache = true

[source_overrides]
my-kernel = "../my-kernel"
mlibc = { path = "../mlibc", patch = true, prepare = true }
```

| Key                               | Type           | Default | Description                                                                                            |
| --------------------------------- | -------------- | ------- | ------------------------------------------------------------------------------------------------------ |
| `disable_pruning`                 | bool           | `false` | Skip the automatic store, ledger and build directory pruning that runs after `build` and `install`.     |
| `pkgs.<name>.enable_build_cache`  | bool           | `false` | Keep the build directory of a target package between builds instead of wiping it. See [Caching](caching.md). |
| `tools.<name>.enable_build_cache` | bool           | `false` | Same, for a host tool.                                                                                 |
| `source_overrides.<name>`         | string, a path |         | Replace the source named `<name>` with a local directory. Shorthand for `{ path = "..." }`.            |
| `source_overrides.<name>.path`    | string, a path |         | The local directory to use instead of the declared source.                                             |
| `source_overrides.<name>.patch`   | bool           | `false` | Still apply the source's declared patches to the override.                                             |
| `source_overrides.<name>.prepare` | bool           | `false` | Still run the source's `prepare` script on the override.                                               |

!!! note

    Source overrides are the main tool for working on a component: point one at your checkout and
    Chariot builds from it, hashing the directory contents so edits are picked up automatically.
    See [Sources](sources.md#working-on-a-source-locally).

### JSON instead of TOML

If you generate the local config from a script, JSON may be easier to produce:

```console
$ chariot --local-config-format json build --arch x86_64 my-package
```

This changes the default path to `.chariot.json`. `--local-config <path>` overrides the path
regardless of format.
