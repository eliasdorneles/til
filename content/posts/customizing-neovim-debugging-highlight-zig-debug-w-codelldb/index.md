---
title: "Debugging Zig: with lldb and inside Neovim"
date: "2024-08-10T00:22:15+02:00"
tags:
  - Neovim
  - debugging
  - Zig
---

Today I learned how to use a debugger to debug Zig programs.

### The lldb way (gdb style)

First, I learned how to use [lldb](https://lldb.llvm.org/), the debugger
provided by the LLVM project.

It works nicely, it's simple enough, it feels a lot like gdb: you run `lldb
$EXECUTABLE_FILE`, and that will open a shell debugger that you can use like:

First, add a breakpoint to the main function with command: `b main`.

Then, start running the program with command `r` (stands for run).

Then, you can use `n` to go to the next line (step over) and `s` to step into
the function in the current line.

It also has a `gui` command that will display a user interface GUI-like, where
you can still do `n` to go to the next line and `s` to step into, but it's more
responsive because no need to hit `<Enter>` after each command.

Here is what it looks like:

{{< img src="debug-lldb.png" alt="Debugging a Zig program with LLDB - Screenshot" size-method="Full" size-format="600x400 webp" position="center" >}}

Okay, so that works, can we do better?

### The nvim-dap way + codelldb

So I've started using [nvim-dap](https://github.com/mfussenegger/nvim-dap) and
[nvim-dap-ui](https://github.com/rcarriga/nvim-dap-ui) recently, and I'm
hooked! These are plugins that use the [Debug Adapter
Protocol](https://microsoft.github.io/debug-adapter-protocol/), which is
basically a protocol for editors/IDEs to talk to debugger tools.

In order to debug Zig code with nvim-dap, you need to configure an adapter.
For Linux, there are mainly two options:

First option: a simpler adapter that uses `lldb` directly: this kinda works, but when attempting to debug a Zig program that reads commands from the standard input, I couldn't figure how to feed it.

Second option: using the `codelldb` adapter, which is actually a VS Code
extension that wraps a custom version of `lldb`, and this works better!

To install `codelldb`, well, you actually gotta install VS code or
[VSCodium](https://vscodium.com/) beforehand, and then install the
[CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb)
extension on it. Once you have it, then you can find out the path to the
`codelldb` executable and put it in your $PATH.

In my system, Ubuntu 22.04, I used VSCodium, and I found the `codelldb`
executable in
`~/.vscode-oss/extensions/vadimcn.vscode-lldb-1.10.0-universal/adapter/codelldb`,
so I basically just ran in my terminal:

```
cd ~/bin
ln -s ~/.vscode-oss/extensions/vadimcn.vscode-lldb-1.10.0-universal/adapter/codelldb
```

Anyway, once you have that, then you can setup the adapter for nvim-dap to use,
here's what I put in my Neovim config (Lua code):

```lua
local dap = require("dap")

-- configure codelldb adapter
dap.adapters.codelldb = {
    type = "server",
    port = "${port}",
    executable = {
        command = "codelldb",
        args = { "--port", "${port}" },
    },
}

-- setup a debugger config for zig projects
dap.configurations.zig = {
    {
        name = 'Launch',
        type = 'codelldb',
        request = 'launch',
        program = '${workspaceFolder}/zig-out/bin/${workspaceFolderBasename}',
        cwd = '${workspaceFolder}',
        stopOnEntry = false,
        args = {},
    },
}
```

Here is what looks like using it:

{{< img src="debug_nvim_dap_codelldb.png" alt="Debugging a Zig program with Neovim DAP + adapters - Screenshot" size-method="Full" size-format="600x400 webp" position="center" >}}

[Full config available here](https://github.com/eliasdorneles/dotfiles/blob/31b6a1c5f2c84ea8a4bd6842f29e79e963d4268e/config/nvim/lua/settings.lua).
