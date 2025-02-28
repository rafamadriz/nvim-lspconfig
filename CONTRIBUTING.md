## Requirements

- [Neovim](https://neovim.io/) 0.6 or later
- Lint task requires [luacheck](https://github.com/luarocks/luacheck#installation) and [stylua](https://github.com/JohnnyMorganz/StyLua). If using nix, you can use `nix develop` to install these to a local nix shell.
- Documentation is generated by `scripts/docgen.lua`.
  - Only works on linux and macOS

## Scope of lspconfig

The point of lspconfig is to provide the minimal configuration necessary for a server to act in compliance with the language server protocol. In general, if a server requires custom client-side commands or  off-spec handlers, then the server configuration should be added *without* those in lspconfig and receive a dedicated plugin such as nvim-jdtls, nvim-metals, etc.

## Pull requests (PRs)

- To avoid duplicate work, create a draft pull request.
- Avoid cosmetic changes to unrelated files in the same commit.
- Use a [feature branch](https://www.atlassian.com/git/tutorials/comparing-workflows) instead of the master branch.
- Use a **rebase workflow** for small PRs.
  - After addressing review comments, it's fine to rebase and force-push.

## Adding a server to lspconfig

The general form of adding a new language server is to start with a minimal skeleton. This includes populated the `config` table with a `default_config` and `docs` table.

When choosing a server name, convert all dashes (`-`) to underscores (`_`) If the name of the server is a unique name (`pyright`, `clangd`) or a commonly used abbreviation (`zls`), prefer this as the server name. If the server instead follows the pattern x-language-server, prefer the convention `x_ls` (`jsonnet_ls`). 

`default_config` should include, at minimum the following:
* `cmd`: a list which includes the executable name as the first entry, with arguments constituting subsequent list elements (`--stdio` is common).
Note that Windows has a limitation when it comes to directly invoking a server that's installed by `npm` or `gem`, so it requires additional handling.

```lua
cmd = { 'typescript-language-server', '--stdio' }
```

* `filetypes`: a list for filetypes a 
* `root_dir`: a function (or function handle) which returns the root of the project used to determine if lspconfig should launch a new language server, or attach a previously launched server when you open a new buffer matching the filetype of the server. Note, lspconfig does not offer a dedicated single file mode (this is not codified in the spec). Do not add `vim.fn.cwd` or `util.path.dirname` in `root_dir`. A future version of lspconfig will provide emulation of a single file mode until this is formally codified in the specification. A good fallback is `util.find_git_ancestor`, see other configurations for examples.

Additionally, the following options are often added:

* `init_options`: a table sent during initialization, corresponding to initializationOptions sent in [initializeParams](https://microsoft.github.io/language-server-protocol/specifications/specification-3-17/#initializeParams) as part of the first request sent from client to server during startup.
* `settings`: a table sent during [`workspace/didChangeConfiguration`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-17/#didChangeConfigurationParams) shortly after server initialization. This is an undocumented convention for most language servers. There is often some duplication with initOptions.

An example for adding a new language server is shown below for `pyright`, a python language server included in lspconfig:

```lua
local util = require 'lspconfig.util'

local root_files = {
  'pyproject.toml',
  'setup.py',
  'setup.cfg',
  'requirements.txt',
  'Pipfile',
  'pyrightconfig.json',
}

local function organize_imports()
  local params = {
    command = 'pyright.organizeimports',
    arguments = { vim.uri_from_bufnr(0) },
  }
  vim.lsp.buf.execute_command(params)
end

return {
  default_config = {
    cmd = { 'pyright-langserver', '--stdio' },
    filetypes = { 'python' },
    root_dir = util.root_pattern(unpack(root_files)),
    single_file_support = true,
    settings = {
      python = {
        analysis = {
          autoSearchPaths = true,
          useLibraryCodeForTypes = true,
          diagnosticMode = 'workspace',
        },
      },
    },
  },
  commands = {
    PyrightOrganizeImports = {
      organize_imports,
      description = 'Organize Imports',
    },
  },
  docs = {
    description = [[
https://github.com/microsoft/pyright

`pyright`, a static type checker and language server for python
]],
  },
}
```

## Commit style

lspconfig, like neovim core, follows the [conventional commit style](https://www.conventionalcommits.org/en/v1.0.0-beta.2/) please submit your commits accordingly. Generally commits will be of the form:

```
feat: add lua-language-server support
fix(lua-language-server): update root directory pattern
docs: update README.md
```

with the commit body containing additional details.

## Lint

PRs are checked with [luacheck](https://github.com/mpeterv/luacheck), [StyLua](https://github.com/JohnnyMorganz/StyLua) and [selene](https://github.com/Kampfkarren/selene). Please run the linter locally before submitting a PR:

    make lint

## Generating docs

GitHub Actions automatically generates `server_configurations.md`. Only modify `scripts/README_template.md` or the `docs` table in the server config Lua file. Do not modify `server_configurations.md` directly.

To preview the generated `server_configurations.md` locally, run `scripts/docgen.lua` from
`nvim` (from the project root):

    nvim -R -Es +'set rtp+=$PWD' +'luafile scripts/docgen.lua'
