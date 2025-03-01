+++
title = "My Neovim Setup"
date = "2025-03-01T17:17:00Z"
description = "Post covering my neovim setup, plugin choices and how to use it."

tags = []
+++

## Intro

In this post I'm going to over my custom neovim configuration, explain how it works, what's included, and how you can use it. This post assumes you already have neovim installed on your machine, or at least know how to do so, and that you have a basic understanding of vim. If you don't, I'd recommend checking out the sources section at the bottom of the post to learn more.

I've been using vim for a long time. It started in university when I needed an upgrade from trusty old nano in the terminal. Then over the years I've been using a vim emulator in IntelliJ and VSCode. Recently, I became envious of people like the primeagen using their customised neovim setups to get stuff done. It looked cool, so I decided I wanted to have a cool setup neovim setup too!

For those that don't know, neovim is a fork of vim that uses lua for extending the base functionality. There are many neovim 'distros' out there, which are pre-defined configurations that include various plugins and themes to give you a seamless out of the box riced neovim experience. A popular one is LazyVim. My config borrows aspects from Lazyvim but it is nonetheless a 'from scratch' configuration. I wanted to build my own so that I could better understand how it all worked, and also to have a config that perfectly suits my needs without any additional bloat.

## Getting started

The `init.lua` file located in the base `~/.config/nvim` directory is required for all custom nvim configs. In my case, I've modularised it so that it imports the files that contain my actual config.
To use this config, simply copy the contents of the `nvim/` directory found in the [repo](https://github.com/mj3f/dotfiles) to your .config directory by running `cp -r dotfiles/nvim ~/.config` (this cp command may need tweaking to fit your choice of OS but you get the idea). Once this is done, simply run `nvim` and everything should install automatically.

### init.lua

The contents of my `init.lua` file look like:

```lua
require("core.keymaps")
require("core.plugins")
require("core.plugin_config")
```

This simply imports the lua files located within the `core/` directory. `keymaps.lua` is where you'll find all my custom keymap overrides for vim (note this doesn't include any custom keymappings for plugins). `plugins.lua` is where all the plugins are defined, and `plugin_config/` is a directory containing several lua files for each plugin installed, used to initialise the plugin(s) with custom settings.

## Setting custom key mappings

Custom key maps are located in the `lua/core/keymaps.lua` file.
As neovim uses lua rather than vimscript, you need to use 'meta-accessors' to access the underlying vim apis.
refer to [this](https://github.com/nanotee/nvim-lua-guide?tab=readme-ov-file#using-meta-accessors) for more information, but the main two accessors are `vim.o` or `vim.opt`, and they're used to 'set' stuff in vim.
`vim.g` is similar except it's applies globally, rather than at the buffer level. This is usually used to set values for settings used by the plugins you import.

## Adding plugins

Lazy.nvim is a package manager used to install packages for your neovim config. There are other package managers out there (I've used Packer in the past), but Lazy.nvim seems to be the most feature-rich and popular these days, so you shouldn't have issues finding install instructions for plugins in Lazy.nvim.
Adding plugins can be achieved by adding them to the `lua/core/plugins.lua` file, under the 'plugins' table towards the bottom of the file.

e.g. (this example demonstrates how to install the 'gruvbox' neovim theme)

```lua
local plugins = {
  { "ellisonleao/gruvbox.nvim", priority = 1000 , config = true, opts = ...},
}
```

The first part is the name of the repo for the plugin in github, some plugin definitions will contain just the name, others may need to define configurations and/or dependencies.
Once the plugin has been added, if you run the `nvim` command it should automatically install the plugin. Afterwards make sure to run the vim command `:checkhealth name-of-plugin` to check the status of the plugin post-install, and whether you need to do any other setup to it.

Lastly, you need to actually load the plugin in order to use it, this is done by calling `require("gruvbox").setup()`. This 'imports' the plugin, much like how you would import packages in various programming languages, and `.setup()` initialises it.
In my neovim config, each plugin has it's own lua file, located in `lua/core/plugin_config`.
If a plugin requires further setup, such as tweaking its default options, you can define a local variable to easily access the plugin, e.g. `local config = require("gruvbox") \n config.setup({...})`

The `:Lazy` vim command opens a nice TUI window to show you a list of all installed plugins, and an easy way to update them.

## Adding language support

The Mason plugin is used to manage any installed LSP (Language Server Protocol) and Linter/formatters. The TUI can be accessed by running the `:Mason` command.
To add a new LSP, you need to add the name of it to the ensure_installed table located within the `lua/core/plugin_config/lsp_config.lua` file. After that is done, you need to setup the LSP at the bottom of the file by adding:

```lua
lspconfig.cssls.setup {
  capabilities = capabilities
}
```

(capabilities ensures that code completions for the newly added language are supported)

To install linters, like prettier, it is a slightly different process. You can install them using `:Mason`, but to configure and initialise them, you need to do the setup within the `conform.lua` file within the `plugins_config`.

## Installed Plugins

Below is a brief summary of each installed plugin at the time of this post (mainly to remind me what they do!)

- Gruvbox is the theme of choice
- telescope for fuzzy finding of files within the directory
- nvim-treesitter for syntax highlighting and colour support for defined programming languages?
- nvim-tree for a file directory window popup
- lualine for a cool bar at the bottom of the terminal window
- gitsigns for showing when code has been added/removed/modified between commits
- git-blame to blame myself when something goes wrong.
- nvim-cmp & cmp-nvim-lsp for language support.
- mason for managing LSPs.
- LuaSnip for code snippets.
- which-key as a helpful legend for all defined keymaps.
- dashboard-nvim for a cool neovim start screen.
- conform for setting up linters.
- harpoon for quickly switching between frequently used files in the current session.

## Source (and other misc bits)

I followed (the free part) of [this course from typecraft](https://typecraft.dev/neovim-for-newbs/) to create my neovim configuration. It is a paid course but the free section should be more than enough for you to get started with building your own config.
Learn more about neovim [here](https://neovim.io/)
If you're new to vim, I recommend [this youtube playlist](https://www.youtube.com/playlist?list=PLm323Lc7iSW_wuxqmKx_xxNtJC_hJbQ7R) from 'ThePrimeagen' to get started.
