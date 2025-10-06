---
title: Install Neovim Plugins Without a Plugin Manager
date: 2025-10-05
description: A guide on how to install Neovim plugins without a plugin manager.
---

Installing Neovim plugins is simple, but it does need a bit of extra code. You
can do this if you do not need a plugin manager; in such a case, you only have
fewer plugins. First, I will explain how plugins and the configuration are
searched and loaded by the Neovim runtime path and then the steps on how to
install, load, and set up the plugins.

## Understanding About Neovim Runtime Path

Before going to the steps, let me tell you what is `runtimepath` in Neovim and
how it can recognize the plugins.

`runtimepath` is a list of Neovim directory paths that will be automatically
searched and loaded when you open Neovim. I will only list of two important path
you need to know here that defined by Neovim runtime:

- `$XDG_CONFIG_HOME/nvim` or `~/.config/nvim`. It is a place to store your
  Neovim configuration.
- `$XDG_DATA_HOME/nvim/site` or `~/.local/share/nvim/site`. It is a place to
  store your Neovim plugins.

I will list a few _global_ directories that you may be familiar with, where the
Neovim runtime path will search for. Do not get confused by what the meaning
of "global directories" is. I will explain it after it, but here are some of a
few:

- `colors/` is a place for color scheme.
- `doc/` is a place for documentation or for a `:help` page.
- `ftplugin/` is a place to configure Neovim that will be triggered on a
  specific file type.
- `lua/` is a place to store Lua modules.

There you can create the directories and store some Lua files based on their
usages. For an example, this is the structure for Neovim configuration in
`$XDG_CONFIG_HOME/nvim`:

```
nvim/
├── colors
│   └── dark_moon.lua
├── doc
│   └── help_page.txt
├── ftplugin
│   ├── go.lua
│   └── javascript.lua
├── init.lua
└── lua
    ├── init.lua
    └── my-module
        └── example_module.lua
```

The reason why it is called a "global directories" is that the runtime path not
only searched those directories from your Neovim configuration path, but also
from `$XDG_DATA_HOME/nvim/site`, where the plugins will be stored in either
`$XDG_DATA_HOME/nvim/site/pack/*/start` or `$XDG_DATA_HOME/nvim/site/pack/*/opt`.
The differences between `start` and `opt` at the end of the path will be
explained in the [steps part](#the-practical-steps).

But what you need to know is when store plugins like the Catppuccin theme at
`start`, the path will be like
`$XDG_DATA_HOME/nvim/site/pack/*/start/catppuccin`. And inside the `catppuccin`
directory, which is [cloned from GitHub](https://github.com/catppuccin/nvim),
there you can see the `colors/`, `doc/`, and `lua/` directory that will be
searched and loaded by Neovim runtime, just like the list of directories
I mentioned above.

## The Practical Steps

There are two packages in the Neovim runtime path called `start` and `opt` in
`$XDG_DATA_HOME/nvim/site/pack/*/`. Where as `start` contain plugins that are
automatically loaded when you start Neovim, and `opt` is also contain plugins
but only loaded when needed with `:packadd` or you can simply create your own
custom _event_ in your Neovim configuration to trigger the `:packadd` and this
can behave just like "lazy load" some plugins.

You can use both packages for any plugins, but only one plugin for one package
in place, either `start` or `opt`. I will provide the example of both of those
by installing the [Catppuccin theme](https://github.com/catppuccin/nvim),
[nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter), and
[mini.trailspace](https://github.com/echasnovski/mini.trailspace).

First, create the package directory:

```sh
mkdir -p ~/.local/share/nvim/site/pack/plugins/{start,opt}
```

The package directory will be like this:

```
.local/share/nvim/site/
└── pack
    └── plugins
        ├── opt
        └── start
```

However, you can change the package name to something else other than `plugins`
like `mkdir -p ~/.local/share/nvim/site/pack/vendor/{start,opt}`. But I prefer
`plugins` for simplicity.

Then grab any Neovim plugins you would like to by clonning it to the package
directory. In this example, I will put the Catppuccin theme and nvim-treesitter
to `start`, and mini.trailspace to `opt`:

```sh
cd ~/.local/share/nvim/site/pack/plugins/
git clone --depth=1 https://github.com/catppuccin/nvim start/catppuccin
git clone --depth=1 https://github.com/nvim-treesitter/nvim-treesitter start/nvim-treesitter
git clone --depth=1 https://github.com/echasnovski/mini.trailspace opt/trailspace
```

> **Note**: You can omit the Git `--depth` flag if you want to update the plugin
> frequently by using `git pull` inside the plugin directory. In this case, I
> want to keep the repository size small, and I am not going to update it unless
> there is any problem with the plugin itself.

Plugins from `start` will automatically be loaded when you enter Neovim. You
will also need to load the documentation or help page in Neovim by entering
`:helptags ALL`. After that, you can open the help page for nvim-treesitter
with `:help nvim-treesitter`.

To load plugins from `opt`, you need to do it manually with `:packadd`. For
example, to load mini.trailspace where the directory name in `opt` is
`trailspace`, I add with `:packadd trailspace`. You can use the Tab key after
type `:packadd` to autocomplete a list of available plugins. Then do
`:helptags ALL` to load the documentation page.

After it is loaded, you need to enable the plugins by calling the `setup()`
function in your Neovim `init.lua` configuration. Without it, you will neither
be able to use the commands from the plugin nor will the plugin be usable.

> **Note**: A few of the plugins can run without calling the `setup()` so you
> need to refer to the plugin documentation. Also, the setup and optional
> configuration may be different.

Here is how I do for plugins that are will automatically start:

```lua
local ok, catppuccin = pcall(require, "catppuccin")
if ok then
    catppuccin.setup({ flavour = "mocha" })
    vim.cmd("colorscheme catppuccin")
end

local ok, treesitter = pcall(require, "nvim-treesitter.configs")
if ok then
    treesitter.setup({ highlight = { enable = true } })
end
```

On the code above, the plugin module will be imported by `require()` and wrap
it safely with `pcall()`, then check the result with `ok` variable. If not using
`pcall()`, it will throw an error when you open Neovim if the plugin is not
installed on the package runtime path.

Plugins that are from `opt` can be lazy load, and it take a different approach
from `start` package.

```lua
local group = vim.api.nvim_create_augroup("user", { clear = true })
vim.api.nvim_create_autocmd("BufWrite", {
    group = group,
    pattern = "*",
    callback = function()
        if _G.MiniTrailspace then
            vim.cmd("lua MiniTrailspace.trim()")
        else
            vim.cmd("packadd trailspace | helptags ALL")
            local ok, trailspace = pcall(require, "mini.trailspace")
            if ok then
                trailspace.setup()
                vim.cmd("lua MiniTrailspace.trim()")
            end
        end
    end
}
```

There I created an auto command to trigger the callback function every time I
saved the file with `:w` or what is called a "write buffer" with `BufWrite`
event for all type of file (`pattern` with value `*`). Then check if
`MiniTrailspace` (a global variable from the mini.trailspace plugin) is not
`nil` (meaning the plugin is already loaded), then run the
`MiniTrailspace.trim()` command. Otherwise it will load it first with
`packadd` and `setup()`.

After all plugins are loaded, the commands from the plugins will be available,
such as `:TSInstall`, `:colorscheme catppuccin`,
and `:lua MiniTrailspace.trim()`.

> **Tips**: I do recommend you to only use the `start` package and store all
> plugins there so it will be loaded automatically by Neovim runtime without any
> hassle to write long Lua code just to lazy load the plugins when using
> the `opt` package.

## Conclusion

Having fewer plugins may be a good choice to not use the plugin manager. But
that does not mean plugin manager is useless. You probably will going to need
it someday if you add more a bunch of plugin.

I have been using [lazy.nvim](https://github.com/folke/lazy.nvim) since then,
but now I no longer use it because I only had the nvim-treesitter plugin and
got used to built-in Neovim functionality.

References for this article you can find and read in the Neovim documentation
or built-in help page:

- [`:help runtimepath`](https://neovim.io/doc/user/options.html#'runtimepath')
- [`:help runtime-search-path`](https://neovim.io/doc/user/repeat.html#runtime-search-path)
- [`:help packages`](https://neovim.io/doc/user/repeat.html#packages)
- [`:help packadd`](https://neovim.io/doc/user/repeat.html#%3Apackadd)
- [`:help events`](https://neovim.io/doc/user/autocmd.html#events)
