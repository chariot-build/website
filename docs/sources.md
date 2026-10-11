---
icon: lucide/package-search
---

# Sources

A source is something to build from. Chariot fetches it, verifies it, patches it and optionally runs
a `prepare` script on it, then hands the result to one or more packages as a read-only directory.

## Declaring a source

```lua
local coreutils_source = Source {
    name = "coreutils",
    Archive(
        "https://ftp.gnu.org/gnu/coreutils/coreutils-9.5.tar.xz",
        "cd328edeac92f6a665de9f323c93b712af1858bc2e0d88f3f7100469470a1b8a"
    ),
    patches = {
        "patches/coreutils/0001-fix-build.patch",
    },
}
```

`Source` returns a reference used by packages. Source names must be unique across the project.

| Field          | Type                    | Required | Description                                                                                            |
| -------------- | ----------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| `name`         | string                  | yes      | Unique name. Also the name it appears under in `SOURCES_DIR`.                                          |
| `base`         | source base             | yes      | Where to get the code: `Archive`, `Git` or `Local`.                                                    |
| `patches`      | array of strings, paths | no       | Patch files, relative to the project root, applied in order.                                           |
| `prepare`      | string                  | no       | A [script](scripts.md) run once on the fetched tree, for example to run `autoreconf`.                  |
| `dependencies` | array                   | no       | Dependencies available to the `prepare` script. Same forms as a [package's](packages.md#dependencies). |
| `env`          | map of string to string | no       | Environment variables for the `prepare` script.                                                        |

`name` and `base` may also be given positionally: the first string in the table becomes the name and
the first table becomes the base. The example above uses that shorthand for `base`, and both forms
can be mixed freely.

!!! note

    `dependencies` and `env` only apply to `prepare`. If you set either without a `prepare` script,
    Chariot warns and ignores them.

## Source bases

### `Archive(url, checksum, kind?)`

Downloads an archive and verifies it against a SHA-256 checksum before unpacking.

```lua
Archive(
    "https://www.nasm.us/pub/nasm/releasebuilds/3.02/nasm-3.02.tar.xz",
    "87336eba53b4acfe917424ab5d500d2b0054d9f5148d35c2273ccf2cfb712f0d"
)
```

`kind` is `"tar"` or `"zip"` and is normally inferred from the URL: `.tar`, `.tar.*` and `.tgz` mean
`tar`, `.zip` means `zip`. If the URL does not end in a recognizable extension the inference fails,
so pass `kind` explicitly in that case.

The contents of the archive become the source directly. The wrapper directory archives usually
contain, such as `nasm-3.02/`, is not part of the result.

### `Git(url, revision)`

Clones a repository at an exact revision.

```lua
Git("https://github.com/managarm/mlibc.git", "8101fcc15133eedb26d7e72f3de65e12e67312ae")
```

`revision` must be immutable, so a commit hash or a tag. Branch names are rejected, because a branch
would silently change what gets built while the config stays the same. The clone is shallow.

### `Local(path)`

Uses a directory inside the project root, for example vendored code or a submodule.

```lua
Local("sources/my-bootloader")
```

The directory is copied into the cache and hashed at config evaluation time, so edits are picked up
on the next run.

## Patches

Patch files are read from the project root at config evaluation time and applied in the listed order
with `patch -p1`, each on top of the previous result. Because the patch _contents_ are hashed rather
than the file paths, editing a patch invalidates exactly the sources that use it.

```lua
patches = {
    "patches/gcc/0001-add-my-target.patch",
    "patches/gcc/0002-fix-crtstuff.patch",
}
```

## The `prepare` script

`prepare` runs once per source, after patching, and its output becomes part of the cached source.
Use it for work that belongs to the source rather than to any single package building from it, such
as regenerating a build system.

```lua
Source {
    name = "mlibc",
    Git("https://github.com/managarm/mlibc.git", "8101fcc15133eedb26d7e72f3de65e12e67312ae"),
    dependencies = { "autoconf", "automake", "libtool" },
    prepare = [[
        cd "${SOURCE_DIR}"
        ./bootstrap
    ]],
}
```

The script starts in the source tree with `SOURCE_DIR` pointing at it, and unlike during a package
build, the tree is writable for this step. See [Scripts](scripts.md) for the full environment.

## Using a source

A package names one source as _its_ source, and may depend on any number of others:

```lua
Package {
    name = "gcc",
    source = gcc_source,
    dependencies = { gmp_source },
    -- ...
}
```

Scripts reach the package's own source through `SOURCE_DIR`, and every source dependency through
`SOURCES_DIR`, where each one appears under its declared name. Both are read-only during a package
build. Build systems that insist on writing into their source tree need an out of tree build, or a
`prepare` script that produces the writable state up front.

A source is only fetched when something in the build actually needs it.

## Working on a source locally

While developing a component you usually want to build from your own checkout rather than the
declared archive or git revision. Add a source override to your
[`.chariot.toml`](configuration.md#chariottoml):

```toml title=".chariot.toml"
[source_overrides]
mlibc = "../mlibc"
```

The override replaces the source's base with your directory and, by default, drops the project's
patches and `prepare` script, the assumption being that your checkout already contains them. Opt
back in per source:

```toml
[source_overrides]
mlibc = { path = "../mlibc", patch = true, prepare = true }
```

Overridden directories are hashed like any `Local` source, so your edits invalidate exactly the
packages built from them and nothing else.
