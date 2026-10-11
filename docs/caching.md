---
icon: lucide/database
---

# Caching

Chariot never builds the same thing twice. Every fetch, patch, prepare, build and package step is
keyed on a hash of everything that went into it, and the result is kept in a content addressed
store. A rebuild after a config change only redoes the steps whose inputs actually changed.

## The cache directory

Everything lives in one directory, `.chariot-cache` by default, overridable with `--cache` or
`CHARIOT_CACHE_PATH`:

```text
.chariot-cache/
├── store/        # build results, one directory per hash
├── builddirs/    # per package build directories
├── workdirs/     # scratch space for in-flight work
├── ledger.db     # input hash -> effective hash mapping
├── state.db      # known profiles and the hashes they reference
├── state.toml    # cache format version
└── .gitignore    # written by chariot, ignores everything
```

The cache gitignores itself, so it is safe to leave inside a repository.

!!! warning

    The cache, the [rootfs](rootfs.md) and your project must be on the same filesystem. Chariot
    moves and hardlinks files between them, neither of which works across filesystems.

## How invalidation works

Every step produces a store entry named `<category>-<hash>`:

| Category         | What it holds                                             |
| ---------------- | --------------------------------------------------------- |
| `source.base`    | A fetched archive, git checkout or copied local directory |
| `source.patch`   | The patched tree                                          |
| `source.prepare` | The tree after the `prepare` script ran                   |
| `install`        | What a package's `install` script wrote                   |
| `pkg`            | The finished binary package                               |

Lookups go through two hashes:

1. The **input hash**, computed from the config alone: the scripts, environment, dependencies,
   architecture, prefix, rootfs manifest hash and config format version.
2. The **effective hash**, computed from the actual _content_ of the inputs once they are built: the
   real bytes of the sources and dependency packages.

The ledger maps input hash to effective hash, and the store is keyed on the effective hash. The
indirection is what makes rebuilds cheap: if a dependency is rebuilt but produces byte identical
output, everything downstream still resolves to the same effective hash and nothing else rebuilds.

Packaging is split from building for the same reason. A package's `install` result depends on its
content hash, while the final `pkg` depends on the metadata hash (name, version, revision, runtime
dependencies) plus the installed tree, so bumping only a `version` or `revision` repackages without
recompiling.

### What causes a rebuild

| Change                                                                    | Effect                                                                         |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| A `configure`, `build`, `install` or `prepare` script                     | That package or source rebuilds                                                |
| A package's `env`, native dependencies or dependency set                  | That package rebuilds                                                          |
| A source URL, checksum, revision, patch content, or local directory       | That source is re-fetched, dependants rebuild                                  |
| `version`, `revision` or `runtime_dependencies`                           | The package is repackaged, not rebuilt                                         |
| `target_prefix`, `global_environment_variables`, `global_native_packages` | Everything rebuilds                                                            |
| `--arch` or `-o` options                                                  | A different [profile](configuration.md#options-and-profiles), built separately |
| The rootfs manifest hash                                                  | Everything rebuilds                                                            |
| A new Chariot release that bumps the config format version                | Everything rebuilds                                                            |

## Profiles and reachability

A profile is one permutation of architecture and options. Chariot records every profile it has
built, along with the hashes that profile's config refers to, in `state.db`. That record is what
makes garbage collection possible: without it, Chariot would have no way to know whether a store
entry belongs to a configuration you still care about.

This is also why a new profile asks for confirmation. See
[Options and profiles](configuration.md#options-and-profiles).

## Reclaiming space

```console
$ chariot cache gc        # drop everything no known profile refers to
$ chariot cache purge     # drop everything
$ chariot cache list-ledger
```

`gc` re-evaluates the config for every known profile, collects the hashes still reachable, and
removes the rest from the store and ledger. `purge` empties both unconditionally; the next build
starts from scratch but does not need to re-download the rootfs.

After every `build` and `install`, Chariot also prunes automatically, which keeps the store from
accumulating results for packages you have deleted from the config. Set `disable_pruning = true` in
[`.chariot.toml`](configuration.md#chariottoml) if you are switching between branches a lot and
would rather keep the old results around.

Entries currently in use are locked and are never removed, so pruning is safe to run while another
Chariot process is building.

## Build directories

Build directories live in `builddirs/`, one per platform, architecture and package name. By default
a package's build directory is wiped before each build, because a stale directory is a common source
of builds that only work on your machine.

For a package you iterate on constantly, such as a kernel or a large C++ tree, that wipe is
expensive. Opt into keeping it:

```toml title=".chariot.toml"
[pkgs.my-kernel]
enable_build_cache = true

[tools.gcc]
enable_build_cache = true
```

With this enabled, `make` sees its old object files and only rebuilds what changed. This is a local
development convenience, which is why it lives in the local config: results built this way are not
guaranteed to be reproducible.

A kept build directory is also what `chariot exec -b <dest>=<package>` mounts, which makes it
useful for inspecting a failed build.

## Disk layout

Two things keep a distribution build from filling the disk:

- **overlayfs.** Sources are presented to a build as a stack of layers (base, patched, prepared)
  rather than as a merged copy, and host tools and native packages are stacked onto the rootfs the
  same way.
- **Hardlinks.** Files shared between rootfs package sets are hardlinked together instead of
  stored twice, which matters because sets overlap heavily.
