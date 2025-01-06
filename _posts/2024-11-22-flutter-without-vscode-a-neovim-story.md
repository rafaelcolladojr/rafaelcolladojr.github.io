---
layout: post
title: "Flutter without Vscode: A Neovim Story"
category: tutorials
tags: flutter dart neovim tutorial plugin lsp debug
---

Google's Flutter framework is rapidly gaining popularity among developers for enabling the creation of beautiful, natively-compiled applications for mobile, web, and desktop from a single codebase. However, the abundance of Flutter tutorials focused on editors like VSCode and Android Studio can be discouraging for developers who use or are interested in **Neovim**.  
<br>
Let's put that to an end.  

## Why Neovim?
**Neovim** is the terminal text editor that might finally earn your engineer father's approval. It's a modern take on **Vim**, a highly configurable and efficient editor celebrated for its powerful modal editing capabilities and mouse-free philosophy.  
Between its lightweight, resource efficient nature and keyboard-centric workflow, **Neovim** can easily make Flutter's already fast UI workflow blazingly fast.  
<br>
It's also written and configured in **Lua**.  

## The setup
To grant **Neovim** all the bells and whistles of a VSCode Flutter experience, we'll take advantage of two key plugins: 
[dart-vim-plugin](https://github.com/dart-lang/dart-vim-plugin) and [flutter-tools.nvim](https://github.com/nvim-flutter/flutter-tools.nvim).

> **Note**: This tutorial assumes you have a basic **Neovim** configuration in place. For help setting **Neovim** up to this point, consider using [*kickstart.nvim*](https://github.com/nvim-lua/kickstart.nvim) to get a basic IDE experience set up.  

### dart-vim-plugin
`dart-vim-plugin` adds important Dart language support to Vim. It offers features such as syntax highlighting, code indentation, and basic code completion.
Start by importing the plugin via your preferred plugin manager, I use [lazy.nvim](https://github.com/folke/lazy.nvim):  
```lua
require("lazy").setup({

  // ... Among your other plugins

  'dart-lang/dart-vim-plugin',
})
```
We can now enable this plugin's features via the following global variables:  
```lua
vim.g.dart_style_guide = 2
vim.g.dart_format_on_save = 1
vim.g.dart_trailing_comma_indent = true
```
`dart_style_guide` sets the tab width used when auto-formatting. In our case, we mimic the VSCode experience by setting it to two spaces.  
`dart_format_on_save` is self-explanatory; it triggers the Dart auto-formatter upon writing the buffer.  
`dart_trailing_comma_indent`, my favorite, ensures proper indentation when adding a trailing comma to comma-separated lists like function parameters and argument listings.  

### flutter-tools.nvim
`flutter-tools.nvim` is a plugin that provides all the functionality we know and love from the Flutter platform: 
hot reload/restart, device management, debugger support, and integration with the Dart analysis server for real-time code analysis.  
The setup for this plugin is a bit more involved, but nothing we can't handle.  
Just like before, we first add the plugin:  
```lua
require("lazy").setup({

  // ... Among your other plugins

  'dart-lang/dart-vim-plugin',
  {
    'akinsho/flutter-tools.nvim',
    dependencies = {
      'nvim-lua/plenary.nvim',
      'stevearc/dressing.nvim',
    },
  },
})
```
> The plugin depends on **plenary.nvim** and **dressing.nvim**, so make sure to include them as `dependencies`.  

Because `flutter-tools.nvim` directly works with the Dart language server, we can (and should) avoid configuring dartls in our lspconfig setup.
For a near-VSCode experience, setup the plugin with the following snippet:  
```lua
require('flutter-tools').setup({
  decorations = {
    statusline = {
      app_version = true,
      device = true,
    },
  },
  widget_guides = {
    enabled = true,
  },
  closing_tags = {
    highlight = 'Comment',
    prefix = '//',
    enabled = true,
  },
  lsp = {
    color = {
      enabled = true,
      background = true,
      foreground = false,
      virtual_text = true,
      virtual_text_str = "■",
    },
    settings = {
      showTodos = true,
      completeFunctionCalls = true,
      enableSnippets = true,
    },
  },
})
```
`decorations` controls what information is shown in the buffer status line.  

`widget_guides` draws the widget guide lines indicating the parent-child relationships within the buffer. While still experimental, this feature can come in handy!  

`closing_tags` determines the appearance of closing tags - the "comments" after the closing braces of a widget. This setup ensures the 'Comment' highlight group is used when rendering them.  

`lsp` describes how the built-in Dart LSP configuration behaves. In color we've enabled virtual text messages for warnings, syntax errors, etc. And in settings, we're enabling features like TODO diagnostics, auto-completing function calls with required parameters, and including snippets like
`stless -> StatelessWidget…` in code completion.  

With that configured, we now have access to some useful commands, only four of which we really need to know for now:  

`:FlutterRun` Starts the Flutter application in the current directory  
`:FlutterQuit` Quits the currently running Flutter application  
`:FlutterReload` Performs a Hot Reload on the currently running app  
`:FlutterRestart` Performs a Hot Restart on the currently running app  

Now while you can type out these commands whenever you need them, part of the **Neovim** fun is turning things into convenient keybindings:  
```lua
vim.keymap.set('n', '<leader>ff', ':FlutterRun<CR>')
vim.keymap.set('n', '<leader>fq', ':FlutterQuit<CR>')
vim.keymap.set('n', '<leader>fr', ':FlutterReload<CR>')
vim.keymap.set('n', '<leader>fR', ':FlutterRestart<CR>')
```

## Debugging

If you're like me and exclusively develop in Flutter using a debugger, I've got you covered. To achieve this, we'll need the help of two more plugins:  
[nvim-dap](https://github.com/mfussenegger/nvim-dap) and [nvim-dap-ui](https://github.com/rcarriga/nvim-dap-ui).  

By connecting these two plugins to our beloved `flutter-tools.nvim` we can achieve a pretty smooth Flutter debugging experience:  

Start by configuring the plugins, we'll take it straight from their README:  

```lua
require('dapui').setup()
local dap, dapui = require("dap"), require("dapui")
dap.listeners.after.event_initialized["dapui_config"] = function()
 dapui.open()
end
dap.listeners.before.event_terminated["dapui_config"] = function()
 dapui.close()
end
dap.listeners.before.event_exited["dapui_config"] = function()
 dapui.close()
end

local sign = vim.fn.sign_define
sign("DapBreakpoint", { text = "●", texthl = "DapBreakpoint", linehl = "", numhl = ""})
sign("DapBreakpointCondition", { text = "●", texthl = "DapBreakpointCondition", linehl = "", numhl = ""})
sign("DapLogPoint", { text = "◆", texthl = "DapLogPoint", linehl = "", numhl = ""})

vim.keymap.set('n', '<leader>db', ':lua require("dap").toggle_breakpoint()<CR>')
vim.keymap.set('n', '<leader>dB', ':lua require("dap").set_breakpoint(vim.fn.input("Breakpoint Condition: "))<CR>')
vim.keymap.set('n', '<leader>dd', ':lua require("dap").continue()<CR>')
vim.keymap.set('n', '<leader>do', ':lua require("dap").step_over()<CR>')
vim.keymap.set('n', '<leader>di', ':lua require("dap").step_ito()<CR>')
```

Then enable debugging via dap in our `flutter-tools.nvim` configuration:  

```lua
require('flutter-tools').setup {
  decorations = {
    statusline = {
      app_version = true,
      device = true,
    },
  },
  widget_guides = {
    enabled = false,
  },
  closing_tags = {
    highlight = 'Comment',
    prefix = '//',
    enabled = true,
  },
  dev_log = {
    enabled = false,
  },
  lsp = {
    color = {
      enabled = true,
      background = true,
      foreground = false,
      virtual_text = true,
      virtual_text_str = "■",
    },
    settings = {
      showTodos = false,
      completeFunctionCalls = true,
      enableSnippets = true,
    },
  },
  debugger = {
    enabled = true,
    run_via_dap = true,
    exception_breakpoints = {},
    register_configurations = function(_)
      require('dap').configurations.dart = {}
      require("dap.ext.vscode").load_launchjs()
    end,
  },
}
```

> Dap integration is enabled by including the `debugger` map in our configuration object.

As a bonus, this setup allows us to use our pre-existing VSCode `launch.js` configurations from within **Neovim**. Running the `:FlutterRun` command, or simply using our new **\<leader\>ff** keybinding, will display a list of our VSCode run configurations to select from.  

Lastly, the `nvim-dap` README provides some handy keybindings to navigate the debugger - things like breakpoint management and code traversal:  

```lua
vim.keymap.set('n', '<leader>db', ':lua require("dap").toggle_breakpoint()<CR>')
vim.keymap.set('n', '<leader>dB', ':lua require("dap").set_breakpoint(vim.fn.input("Breakpoint Condition: "))<CR>')
vim.keymap.set('n', '<leader>dd', ':lua require("dap").continue()<CR>')
vim.keymap.set('n', '<leader>do', ':lua require("dap").step_over()<CR>')
vim.keymap.set('n', '<leader>di', ':lua require("dap").step_into()<CR>')
```

## Conclusion

When I began my Flutter career, I was afraid I'd be permanently forced to work in VSCode and ditch my naive fantasy of living in the terminal.  

However, thanks to the rich community overlap between Flutter and **Neovim**, I was able to smoothly transition from VSCode to **Neovim** with just four plugins. And now so can you!
