---
icon: lucide/terminal
---

# CLI Reference

```text
chariot [global options] <command> [command options]
```

Run `chariot <command> --help` for the same information from the binary itself.

## Global options

These come before the command name.

| Option                           | Default         | Description                                                  |
| -------------------------------- | --------------- | ------------------------------------------------------------ |
| `--local-config <path>`          | `.chariot.toml` | Path to the [local config](configuration.md#chariottoml).    |
| `--local-config-format <format>` | `toml`          | `toml` or `json`. JSON defaults the path to `.chariot.json`. |
| `--version`                      |                 | Print the Chariot version.                                   |
| `--help`                         |                 | Print help.                                                  |

## Shared option groups

Most commands share these.

### Config options

Used by `install`, `exec`, `lookup`, `build` and `lsp`.

| Option                 | Environment                | Default               | Description                                                                        |
| ---------------------- | -------------------------- | --------------------- | ---------------------------------------------------------------------------------- |
| `--arch <arch>`        | `CHARIOT_ARCH`             | required              | Target architecture.                                                               |
| `--base-config <path>` | `CHARIOT_BASE_CONFIG_PATH` | `chariot_config.toml` | The project's base config. Its directory is the project root.                      |
| `--cache <path>`       | `CHARIOT_CACHE_PATH`       | `.chariot-cache`      | Cache directory.                                                                   |
| `-o`, `--option <K=V>` | `CHARIOT_OPTIONS`          | none                  | User defined option, readable as `chariot.options`. Repeatable or comma separated. |
| `--allow-new-profiles` |                            | off                   | Accept a previously unseen architecture and options combination without prompting. |

### Build options

Added by `install`, `exec`, `build` and `lsp`.

| Option                    | Environment           | Default           | Description                                                                |
| ------------------------- | --------------------- | ----------------- | -------------------------------------------------------------------------- |
| `--rootfs <path>`         | `CHARIOT_ROOTFS_PATH` | `.chariot-rootfs` | Rootfs directory, created on first use.                                    |
| `-j`, `--parallelism <n>` |                       | available cores   | Total job budget shared by all tasks, exposed to scripts as `PARALLELISM`. |
| `--keep-going`            |                       | off               | Keep building unrelated packages after a failure instead of stopping.      |

### Environment variables

Every path related flag can be set from the environment, which is convenient in a shell profile or
CI job:

| Variable                   | Equivalent flag | Default               |
| -------------------------- | --------------- | --------------------- |
| `CHARIOT_ARCH`             | `--arch`        | required              |
| `CHARIOT_OPTIONS`          | `--option`      | none                  |
| `CHARIOT_BASE_CONFIG_PATH` | `--base-config` | `chariot_config.toml` |
| `CHARIOT_CACHE_PATH`       | `--cache`       | `.chariot-cache`      |
| `CHARIOT_ROOTFS_PATH`      | `--rootfs`      | `.chariot-rootfs`     |

## `chariot build`

```text
chariot build [options] <packages>...
```

Builds packages without installing them.

| Option   | Description                                         |
| -------- | --------------------------------------------------- |
| `--tool` | Build host packages (tools) instead of target ones. |

```console
$ chariot build --arch x86_64 --keep-going coreutils bash
$ chariot build --arch x86_64 --tool nasm
```

## `chariot install`

```text
chariot install [options] <packages>... <dest>
```

Builds the given packages and installs them into `<dest>`, which is created if it does not exist.
The last positional argument is the destination.

| Option    | Description                                                 |
| --------- | ----------------------------------------------------------- |
| `--tool`  | Install host packages (tools) instead of target packages.   |
| `--force` | Install even if the package is already present in `<dest>`. |

```console
$ chariot install --arch x86_64 base coreutils bash /mnt/rootfs
```

## `chariot lookup`

```text
chariot lookup [options] <name>
```

Prints the store path of an already built package and exits `0`, or exits `1` if it has not been
built. It never builds anything, which makes it usable in scripts:

```console
$ if path=$(chariot lookup --arch x86_64 coreutils); then echo "built at $path"; fi
```

| Option   | Description                                            |
| -------- | ------------------------------------------------------ |
| `--tool` | Look up a host package (tool) instead of a target one. |

## `chariot exec`

```text
chariot exec [options] <script>
```

Runs a bash script inside a Chariot container. This is the debugging workhorse: it can reproduce a
package's build environment, or assemble an arbitrary one from parts.

| Option                                | Description                                                                                          |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `--build-env <package>`               | Make the environment match the build environment of this target package, including its dependencies. |
| `--pkg <names>`                       | Target packages to build and install into the sysroot. Comma separated.                              |
| `--tool <names>`                      | Host packages (tools) to build and make available. Comma separated.                                  |
| `-p`, `--native-pkg <names>`          | Native rootfs packages to install. Comma separated.                                                  |
| `-b`, `--build-dir <DEST=PACKAGE>`    | Mount a target package's build directory read-only at `DEST`. Repeatable.                            |
| `-m`, `--mount <FROM=TO[:ro][:file]>` | Bind mount a host path into the container. `ro` for read-only, `file` if the source is a file.       |
| `-e`, `--env-var <K=V>`               | Extra environment variables.                                                                         |
| `--cwd <path>`                        | Working directory inside the container. Defaults to `/`.                                             |
| `--stdin`                             | Forward stdin into the container. Needed for an interactive shell.                                   |
| `--no-stdout`                         | Discard the container's stdout.                                                                      |
| `--no-stderr`                         | Discard the container's stderr.                                                                      |

Unlike a real build, the root filesystem is writable here, with changes discarded when the command
exits. That makes it safe to install extra tooling while poking around.

```console
# interactive shell in coreutils' build environment
$ chariot exec --arch x86_64 --build-env coreutils --stdin bash

# inspect the output of a previous build
$ chariot exec --arch x86_64 -b /build=my-kernel 'ls -la /build'

# run something against the target sysroot
$ chariot exec --arch x86_64 --pkg mlibc --tool gcc 'x86_64-chariot-gcc --version'
```

## `chariot lsp`

```text
chariot lsp [options] <lsp> <package>
```

Runs a language server inside a package's build environment, speaking LSP over stdio. See
[Editor Setup](editor-setup.md#a-language-server-for-the-code-you-are-building).

| Option                                  | Description                                                               |
| --------------------------------------- | ------------------------------------------------------------------------- |
| `-m`, `--source-mappings <PATH=SOURCE>` | Map a local directory to a source name so paths line up with your editor. |

## `chariot cache`

```text
chariot cache [--cache <path>] <subcommand>
```

| Subcommand                  | Description                                                           |
| --------------------------- | --------------------------------------------------------------------- |
| `list-ledger`               | Print every ledger record as category, input hash and effective hash. |
| `gc [--base-config <path>]` | Remove everything no known profile refers to.                         |
| `purge`                     | Remove all store and ledger entries.                                  |

See [Caching](caching.md#reclaiming-space).

## `chariot rootfs`

```text
chariot rootfs [--rootfs <path>] <subcommand>
```

| Subcommand                            | Description                                                           |
| ------------------------------------- | --------------------------------------------------------------------- |
| `init [--url <url>] <version> <hash>` | Download and set up a rootfs from a manifest.                         |
| `status`                              | Print the path, manifest URL, version and hash of the current rootfs. |
| `pkgset list`                         | List cached native package sets with their state, stacking and size.  |
| `pkgset cache <packages>...`          | Build a package set up front.                                         |
| `purge`                               | Remove every package set that is not in use.                          |

See [Root Filesystem](rootfs.md).

## `chariot support`

```text
chariot support <subcommand>
```

| Subcommand                         | Description                                                                             |
| ---------------------------------- | --------------------------------------------------------------------------------------- |
| `setup-lsp [--support-dir <path>]` | Generate Lua type definitions and a `.luarc.json`. See [Editor Setup](editor-setup.md). |
| `completions <shell>`              | Print shell completions for `bash`, `zsh`, `fish`, `elvish` or `powershell`.            |
| `licenses`                         | Print Chariot's license and third party notices.                                        |

## Exit codes

| Code | Meaning                                                                      |
| ---- | ---------------------------------------------------------------------------- |
| `0`  | Success.                                                                     |
| `1`  | Failure. For `lookup`, also means not built. The error is printed to stderr. |

When a build fails, Chariot prints which task failed, how many tasks were skipped as a result, and
the captured output of the failing step.
