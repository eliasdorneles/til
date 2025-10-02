---
title: "Setup Love2D and Lua with coc.nvim"
date: "2025-10-02T22:41:19+02:00"
tags:
- neovim
- love2d
- lua
---
I've been coding in Lua lately, mostly writing games using [Love2D](https://www.love2d.org/).

[Neovim](https://neovim.io/) already supports Lua well, as it's the language
used for writing its plugins, but I did install
[coc-lua](https://github.com/josa42/coc-lua) for better completion, inline
documentation and more linting support.

However, I found it annoying that it would ask me everytime I opened a file
inside a Love2d project, asking for confirmation if I wanted to configure it as
a Love2d project. It looks like this:

```
Do you need to configure your work environment as `LÖVE`?
[1]Apply and modify settings, (2)Apply but do not modify settings, (3)Don't show again:
```

I dislike modal dialogs because they disrupt my flow, and this is the TUI
equivalent of it. I think this is an unhappy design choice that some rare
(neo)vim plugins do.

Anyway, to prevent this, open the settings via `:CocConfig`, and add this setting:

```json
    "Lua.workspace.checkThirdParty": "Apply"
```

This will get rid of the dialog, and will always setup Love2D when detecting a
Love2D project.

The [documentation of coc-lua](https://github.com/josa42/coc-lua) only
documents the default boolean value, but that's a setting for the Lua language server.
It supports more values than booleans, [see this section of the docs for all about the checkThirdParty setting](https://luals.github.io/wiki/settings/#workspacecheckthirdparty).
