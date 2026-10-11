---
icon: lucide/code
---

# Editor Setup

## Lua config completion

Chariot can generate type definitions for its Lua API so that
[LuaLS](https://luals.github.io/) gives you completion, hover documentation and diagnostics while
editing `chariot.lua`.

Run this once in your project root:

```console
$ chariot support setup-lsp
```

It writes the definitions to `~/.local/share/chariot/lsp-support` and a `.luarc.json` in the current
directory that points LuaLS at them. The generated config also disables the Lua standard libraries
Chariot does not expose, so a config that calls `os.getenv` is flagged in your editor as you type
rather than only when you run a build.

Use `--support-dir <path>` to put the definitions somewhere else, for example inside the project:

```console
$ chariot support setup-lsp --support-dir .chariot-lsp-support
```

Neither the support directory nor `.luarc.json` should be committed: both are generated, and
`.luarc.json` contains an absolute path specific to your machine. The support directory gitignores
itself.

!!! note

    The definitions are a snapshot of the API of the Chariot version that generated them. Re-run
    `chariot support setup-lsp` after upgrading Chariot.

## A language server for the code you are building

Editing the _source_ of a package is harder than editing the config. The code is built inside a
container, against a sysroot and a cross compiler that do not exist on your host, so a language
server running on your machine has no idea how to parse it.

`chariot lsp` solves that by running the language server inside the package's own build environment:

```console
$ chariot lsp --arch x86_64 clangd my-package
```

`clangd` is currently the only server Chariot knows how to run. Others can be added, so if the
language you work in has one worth supporting, it is worth asking for.

The command speaks the Language Server Protocol over stdin and stdout, so it goes wherever your
editor expects a language server binary. In Neovim:

```lua
vim.lsp.config("clangd", {
    cmd = { "chariot", "lsp", "--arch", "x86_64", "clangd", "my-package" },
})
```

In VS Code, set `clangd.path` to a wrapper script containing the same command.

A few things to know:

- The package must have been built at least once. The language server reads the compilation database
  the build leaves behind in the build directory, so without a build there is nothing to read.
- Running the command builds the package's dependencies first if they are not cached.
- The rootfs manifest must provide the server's binary.

### Mapping sources to your checkout

Inside the container, a source lives wherever Chariot put it, but your editor has the file open at,
say, `~/src/my-package`. Chariot translates paths for the language server automatically, including
for sources that are a stack of layers, but it cannot guess which local directory corresponds to
which source. Tell it with `-m`:

```console
$ chariot lsp --arch x86_64 -m ../my-package=my-package clangd my-package
```

The argument is `<local path>=<source name>`. With a
[source override](configuration.md#chariottoml) configured for the same directory, your edits are
both what the language server sees and what the next build uses.

## Shell completions

See [Installation](installation.md#shell-completions) for generating completions for bash, zsh,
fish, elvish and PowerShell.
