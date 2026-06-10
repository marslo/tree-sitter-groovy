# Tree-sitter parser for Groovy

## Features
- supports most groovy features, including:
   - classes
   - control flow
   - string interpolation
   - closures
   - imports
- tree-sitter queries for
  - highlights
  - indents
  - locals
- rich parse tree to support other extensions like TreeSJ, textobjs (WIP)

## Building & installing

The `Makefile` provides convenience targets for building the parser and installing it where Neovim and the `tree-sitter` CLI expect it.
They wrap `tree-sitter generate` / `tree-sitter build`, so after editing `grammar.js` the generated sources (`src/parser.c`, …) are refreshed automatically.

| TARGET              | WHAT IT DOES                                                                                                                                      | OUTPUT                                                       |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| `make nvim`         | Build the parser Neovim loads. On macOS it also re-signs it (see note below).                                                                     | `./groovy.so`                                                |
| `make nvim-install` | `make nvim` **plus** install into Neovim's runtime parser dir, **plus** refresh the CLI cache (`cli-install`). This is the one to run day-to-day. | `~/.local/share/nvim/site/parser/groovy.so` + CLI cache      |
| `make cli-install`  | Build the parser into the `tree-sitter` CLI cache so `tree-sitter parse` / `highlight` outside this repo use the latest grammar.                  | `~/.cache/tree-sitter/lib/groovy.dylib` (Linux: `groovy.so`) |

```sh
# after changing grammar.js, rebuild & install everywhere:
$ make nvim-install
install -d '~/.cache/tree-sitter/lib'
tree-sitter build -o '~/.cache/tree-sitter/lib/groovy.dylib'
codesign --force --sign - '~/.cache/tree-sitter/lib/groovy.dylib'
~/.cache/tree-sitter/lib/groovy.dylib: replacing existing signature
install -d '~/.local/share/nvim/site/parser'
install -m755 groovy.so '~/.local/share/nvim/site/parser/groovy.so'
codesign --force --sign - '~/.local/share/nvim/site/parser/groovy.so'
~/.local/share/nvim/site/parser/groovy.so: replacing existing signature
```

### nvim nvim-treesitter plugin integration

```vim
" .vimrc
Plug 'nvim-treesitter/nvim-treesitter', { 'branch': 'main', 'do': ':TSUpdate' }
```

```lua
-- init.lua
pcall(function()
  require('nvim-treesitter.parsers').groovy = {
    install_info = {
      path    = '/opt/groovy/tree-sitter-groovy',
      queries = 'queries',
    },
  }
end)
```

- [`:TSUpdateAll`](https://github.com/marslo/dotfiles/blob/main/.config/nvim/lua/config/nvim-treesitter.lua#L146-L171) - updates all parsers and re-build local groovy.so and installs it to Neovim's parser dir.
- [`:TSUpdateGroovy`](https://github.com/marslo/dotfiles/blob/main/.marslo/vimrc.d/functions#L424-L439) - rebuilds the local `groovy.so` and installs it to Neovim's parser dir, without updating other parsers.

Overridable variables:

- `NVIM_PARSER_DIR` — Neovim parser directory (default `~/.local/share/nvim/site/parser`)
- `TS_CACHE_DIR` — tree-sitter CLI cache dir (default `~/.cache/tree-sitter/lib`)
- `TS` — the tree-sitter CLI to use (default `tree-sitter`)

```sh
make nvim-install NVIM_PARSER_DIR=/some/other/parser
```

### Why three different artifacts?

The same grammar is consumed by three independent parsers; updating one does
**not** update the others:

| CONSUMER                                     | FILE IT LOADS                               | REFRESHED BY                                     |
|----------------------------------------------|---------------------------------------------|--------------------------------------------------|
| Neovim (`:InspectTree`, highlighting)        | `~/.local/share/nvim/site/parser/groovy.so` | `make nvim-install`                              |
| `tree-sitter` CLI (run outside the repo)     | `~/.cache/tree-sitter/lib/groovy.dylib`     | `make cli-install` (auto-rebuilt by the CLI too) |
| `tree-sitter test` / parsing inside the repo | `src/parser.c`                              | `tree-sitter generate`                           |

> [!TIP]
> On macOS, `.so` and `.dylib` are the **same** Mach-O format — only the file
> name differs. Neovim always uses the `.so` name on every platform, while the
> CLI uses the OS-native extension (`.dylib` on macOS, `.so` on Linux).

### macOS code signing

`tree-sitter build` emits a *linker-signed* ad-hoc signature that macOS refuses to `dlopen` — Neovim crashes on startup with `SIGKILL` / "Code Signature Invalid".
The `nvim` / `nvim-install` targets therefore re-sign the parser with `codesign --force --sign -` on macOS.
If you build the parser by hand, re-sign it yourself:

```sh
codesign --force --sign - ~/.local/share/nvim/site/parser/groovy.so
codesign --force --sign - ~/.cache/tree-sitter/lib/groovy.dylib
```

## Screenshots
Comparing to [the original groovy parser](https://github.com/Decodetalkers/tree-sitter-groovy)
by @Decodetalkers, here are some screenshots of highlighting:

<img width="300" alt="image" src="https://github.com/murtaza64/tree-sitter-groovy/assets/13615693/137a74cc-2e82-4def-8fd4-67eb88f38221">
<img width="300" alt="image" src="https://github.com/murtaza64/tree-sitter-groovy/assets/13615693/64669396-4366-4bf4-9e92-682ec6cf0dfd">
