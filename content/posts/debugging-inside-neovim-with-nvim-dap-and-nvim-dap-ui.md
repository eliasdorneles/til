---
title: "Debugging inside Neovim with nvim-dap and nvim-dap-ui"
date: "2024-08-08T22:26:37+02:00"
tags:
  - vim
  - neovim
  - debug
  - nvim-dap
  - lua
---

Today I took some time to configure my Neovim with a plugin that I had recently
bumped into: a [the nvim-dap debug
adapter](https://github.com/mfussenegger/nvim-dap) and its UI
[nvim-dap-ui](https://github.com/rcarriga/nvim-dap-ui).

Until recently, the debugger tools I've used most often had been
[pudb](https://github.com/inducer/pudb). Though sometimes for just some basic
inspection I'd sneak an `import IPython; IPython.embed()` in the middle of the
code, which would get me a shell with the local variables there.

Since I've been doing quite a bit of Zig recently, I went looking for a
debugger for it and ended up finding out about Neovim DAP. I've been using vim
for the past 20 years (learned it in 2004), I remember the big jump from vim 6
to vim 7, which supported better mechanisms of completion.

Well, it turns out that in 2024 we can run a full-blown debugger inside it,
with terminal and a funky font that displays nice clickable icons! It's amazing
to see how much progress has happened along these years!

Here is [my configuration](https://github.com/eliasdorneles/dotfiles/blob/0abb8ca94a8e76cdcb01f48807276bd1fe4efa2b/config/nvim/lua/settings.lua#L14-L41):

```lua
-- nvim-dap settings
require("dap-python").setup("python")

-- nvim dap mappings
vim.keymap.set('n', '<F5>', function() require('dap').continue() end)
vim.keymap.set('n', '<F10>', function() require('dap').step_over() end)
vim.keymap.set('n', '<F11>', function() require('dap').step_into() end)
vim.keymap.set('n', '<F12>', function() require('dap').step_out() end)
vim.keymap.set('n', '<M-b>', function() require('dap').toggle_breakpoint() end)

local dap, dapui = require("dap"), require("dapui")
dapui.setup()

-- open Dap UI automatically when debug starts (e.g. after <F5>)
dap.listeners.before.attach.dapui_config = function()
    dapui.open()
end
dap.listeners.before.launch.dapui_config = function()
    dapui.open()
end

-- close Dap UI with :DapCloseUI
vim.api.nvim_create_user_command("DapCloseUI", function()
    require("dapui").close()
end, {})

-- use <Alt-e> to eval expressions
vim.keymap.set({ 'n', 'v' }, '<M-e>', function() require('dapui').eval() end)
```

I feel good with my decision to sticking with vim/neovim, it keeps paying
dividends. Every now and then I will be tempted and try some other tool, just
to realize that my setup with Neovim is superior for me. Well, of course: it's
handmade by me for me! =)
