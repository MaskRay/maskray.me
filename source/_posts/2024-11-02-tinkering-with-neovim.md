---
layout: post
title: Tinkering with Neovim
author: MaskRay
tags: [emacs,neovim]
---

After [migrating from Vim to Emacs](/blog/2015-09-18-conversion-to-emacs) as my primary C++ editor in 2015, I switched from Vim to Neovim for miscellaneous non-C++ tasks as it is more convenient in a terminal.
Customizing the editor with a language you are comfortable with is important.
I found myself increasingly drawn to Neovim's terminal-based simplicity for various tasks.
Recently, I've refined my Neovim setup to the point where I can confidently migrate my entire C++ workflow away from Emacs.

This post explores the key improvements I've made to achieve this transition.
My focus is on code navigation.

<!-- more -->

## Key mapping

I've implemented custom functions that simplify key mappings.

```lua
local function map(mode, lhs, rhs, opts)
  local options = {}
  if opts then
    if type(opts) == 'string' then
      opts = {desc = opts}
    end
    options = vim.tbl_extend('force', options, opts)
  end
  vim.keymap.set(mode, lhs, rhs, options)
end
local function nmap(lhs, rhs, opts)
  map('n', lhs, rhs, opts)
end
local function tmap(lhs, rhs, opts)
  map('t', lhs, rhs, opts)
end
```

I've swapped `;` and `:` for easier access to Ex commands, especially since leap.nvim renders `;` less useful for repeating `ftFT`.
```lua
map({'n', 'x'}, ':', ';')
map({'n', 'x'}, ';', ':')
```

### Cross references

Like many developers, I spend significantly more time reading code than writing it.
Efficiently navigating definitions and references is crucial for productivity.

While the built-in LSP client's `C-]` is functional (see `:h lsp-defaults` `tagfunc`), I found it less convenient.
Many Emacs and Neovim configurations advocate for `gd`.
However, both G and D are placed on the left half of the QWERTY keyboard, making it slow to press them using the left hand.

For years, I relied on `M-j` to quickly jump to definitions.

To avoid a conflict with my recent zellij change (I adopted `M-hjkl` for pane navigation),
I've reassigned `J` to trigger definition jumps.
Although I've lost the original `J` (join lines) functionality, `vJ` provides a suitable workaround.

```lua
nmap('J', '<cmd>Telescope lsp_definitions<cr>', 'Definitions')
nmap('M', '<cmd>Telescope lsp_references<CR>', 'References')
```

After making a LSP-based jump, the jump list can quickly fill with irrelevant entries as I navigate the codebase.
Thankfully, Telescope's LSP functionality sets `push_tagstack_on_edit` to push an entry to the tag stack (see `:h tag-stack`).
To efficiently return to my previous position, I've mapped `H` to `:pop` and `L` to `:tag`.

```lua
nmap('H', '<cmd>pop<cr>', 'Tag stack backward')
nmap('L', '<cmd>tag<cr>', 'Tag stack forward')
```

I've adopted `x` as a prefix key for cross-referencing extensions.
`dl` provide a suitable alternative for `x`'s original functionality.

```lua
nmap('x', '<Nop>')
nmap('xB', '<cmd>CclsBaseHierarchy<cr>')
nmap('xC', '<cmd>CclsOutgoingCalls<cr>', 'callee')
nmap('xD', '<cmd>CclsDerivedHierarchy<cr>')
nmap('xM', '<cmd>CclsMemberHierarchy<cr>', 'member')
nmap('xb', '<cmd>CclsBase<cr>')
nmap('xc', '<cmd>CclsIncomingCalls<cr>', 'caller')
nmap('xd', '<cmd>CclsDerived<cr>')
nmap('xi', '<cmd>lua vim.lsp.buf.implementation()<cr>', 'Implementation')
nmap('xm', '<cmd>CclsMember<cr>', 'member')
nmap('xn', function() M.lsp.words.jump(vim.v.count1) end, 'Next reference')
nmap('xp', function() M.lsp.words.jump(-vim.v.count1) end, 'Prev reference')
nmap('xt', '<cmd>lua vim.lsp.buf.type_definition()<cr>', 'Type definition')
nmap('xv', '<cmd>CclsVars<cr>', 'vars')
```

I utilize `xn` and `xp` to find the next or previous reference.
The implementation, copied from from LazyVim, only works with references within the current file.
I want to enable the `xn` map to automatically transition to the next file when reaching the last reference in the current file.

While using Emacs, I created a hydra with x as the prefix key to cycle through next references.
Unfortunately, I haven't been able to replicate this behavior in Neovim.

```lua
;; This does not work.
local Hydra = require('hydra')
Hydra({
  name = 'lsp xref',
  mode = 'n',
  body = 'x',
  heads = {
    {'n', function() M.lsp.words.jump(1) end},
    {'p', function() M.lsp.words.jump(-1) end},
    { "q", nil, { exit = true, nowait = true } },
  },
})
```

## Movement

I use leap.nvim to quickly jump to specific identifiers (`s{char1}{char2}`), followed by telescope.nvim to explore definitions and references.
Somtimes, I use the following binding:

```lua
nmap('U', function()
  require'hop'.hint_words()
  require'telescope.builtin'.lsp_definitions()
end, 'Hop+definition')
```

## Semantic highlighting

I've implemented rainbow semantic highlighting using ccls.
Please refer to [ccls and LSP Semantic Tokens](/blog/2024-10-20-ccls-and-lsp-semantic-tokens) for my setup.

## Other LSP features

I have configured the `CursorHold` event to trigger `textDocument/documentHighlight`.
When using Emacs, lsp-ui-doc automatically requests `textDocument/hover`, which I now lose.

Additionally, the `LspAttach` and `BufEnter` events trigger `textDocument/codeLens`.

## Window navigation

While I've been content with the traditional `C-w + hjkl` mapping for years, I've recently opted for the more efficient `C-hjkl` approach.

```lua
nmap('<C-h>', '<C-w>h')
nmap('<C-j>', '<C-w>j')
nmap('<C-k>', '<C-w>k')
nmap('<C-l>', '<C-w>l')

tmap('<C-h>', '<cmd>wincmd h<cr>')
tmap('<C-j>', '<cmd>wincmd j<cr>')
tmap('<C-k>', '<cmd>wincmd k<cr>')
tmap('<C-l>', '<cmd>wincmd l<cr>')
```

The keys mirror my pane navigation preferences in tmux and zellij, where I utilize `M-hjkl`.

```
# tmux select pane or window
bind -n M-h if -F '#{pane_at_left}' 'select-window -p' 'select-pane -L'
bind -n M-j if -F '#{pane_at_bottom}' 'select-window -p' 'select-pane -D'
bind -n M-k if -F '#{pane_at_top}' 'select-window -n' 'select-pane -U'
bind -n M-l if -F '#{pane_at_right}' 'select-window -n' 'select-pane -R'
```

```kdl
// zellij M-hjkl
keybinds {
    normal clear-defaults=true {
        ...
        bind "Alt h" { MoveFocusOrTab "Left"; }
        bind "Alt j" { MoveFocus "Down"; }
        bind "Alt k" { MoveFocus "Up"; }
        bind "Alt l" { MoveFocusOrTab "Right"; }
    }
}
```

To accommodate this change, I've shifted my tmux prefix key from `C-l` to `C-Space`.
Consequently, I've also adjusted my input method toggling from `C-Space` to `C-S-Space`.

## Debugging

For C++ debugging, I primarily rely on cgdb. I find it superior to GDB's single-key mode and significantly more user-friendly than LLDB's `gui` command.

```sh
cgdb --args ./a.out args

rr record ./a.out args
rr replay -d cgdb
```

I typically arrange Neovim and cgdb side-by-side in tmux or zellij.
During single-stepping, when encountering interesting code snippets, I often need to manually input filenames into Neovim.
While Telescope aids in this process, automatic file and line updates would be ideal.

Given these considerations, nvim-dap appears to be a promising solution.
However, I haven't yet determined the configuration for integrating rr with nvim-dap.

## Live grep

Telescope's extension telescope-fzf-native is useful.

I've defined mappings to streamline directory and project-wide searches using Telescope's live grep functionality:

```lua
nmap('<leader>sd', '<cmd>lua require("telescope.builtin").live_grep({cwd=vim.fn.expand("%:p:h")})<cr>', 'Search directory')
nmap('<leader>sp', '<cmd>lua require("telescope.builtin").live_grep({cwd=MyProject()})<cr>', 'Search project')
```

Additionally, I've mapped `M-n` to insert the word under the cursor, mimicking Emacs Ivy's `M-n (ivy-next-history-element)` behavior.

## Task runner

I use [overseer.nvim](https://github.com/stevearc/overseer.nvim) to run build commands like `ninja -C /tmp/Debug llc llvm-mc`.
This plugin allows me to view build errors directly in Neovim's quickfix window.

Following LazyVim, I use `<leader>oo` to run builds and `<leader>ow` to toggle the overseer window.
To navigate errors, I use trouble.nvim with the `]q` and `[q` keys.

```lua
nmap('<leader>oo', '<cmd>OverseerRun<cr>')
nmap('<leader>ow', '<cmd>OverseerToggle<cr>')
 
nmap('[q', function()
  if require('trouble').is_open() then
    require('trouble').prev({ skip_groups = true, jump = true })
  else
    local ok, err = pcall(vim.cmd.cprev)
    if not ok then
      vim.notify(err, vim.log.levels.ERROR)
    end
  end
end)
nmap(']q', function()
  if require('trouble').is_open() then
    require('trouble').next({ skip_groups = true, jump = true })
  else
    local ok, err = pcall(vim.cmd.cnext)
    if not ok then
      vim.notify(err, vim.log.levels.ERROR)
    end
  end
end)
```

## Reducing reliance on terminal multiplexer

As <https://rutar.org/writing/from-vim-and-tmux-to-neovim/> nicely summarizes, running Neovim under tmux has some annoyance.
I've been experimenting with reducing my reliance on zellij.
Instead, I'll utilize more Neovim's terminal functionality.

toggleterm.nvim is a particularly useful plugin that allows me to easily split windows, open terminals, and hide them when not in use.

The default command `<C-\><C-n>` (switch to the Normal mode) is clumsy. I've mapped it to `<C-s>` (useless feature [pause transmission](https://en.wikipedia.org/wiki/Software_flow_control), `fwd-i-search` in zsh).

```lua
nmap('<leader>tf', function() require'toggleterm'.toggle(vim.v.count, nil, MyProject(), 'float', nil) end)
nmap('<leader>th', function() require'toggleterm'.toggle(vim.v.count, 10, MyProject(), 'horizontal', nil) end)
nmap('<leader>tv', function() require'toggleterm'.toggle(vim.v.count, 80, MyProject(), 'vertical', nil) end)

tmap('<C-s>', '<C-\\><C-n>')
-- Binding C-/ doesn't work in tmux/zellij
map({'n', 't'}, '<C-/>', '<cmd>ToggleTerm<cr>')
-- This actually binds C-/ in tmux/zellij
map({'n', 't'}, '<C-_>', '<cmd>ToggleTerm<cr>')
```

[neovim-remote](https://github.com/mhinz/neovim-remote) allows me to open files without starting a nested Neovim process.

I use [mini.sessions](https://github.com/echasnovski/mini.sessions) to manage sessions.

## Config switcher

Neovim's [`NVIM_APPNAME`](https://neovim.io/doc/user/starting.html#%24NVIM_APPNAME) feature is fantastic for exploring pre-configured distributions to get inspiration.

## Lua

Neovim embraces Lua 5.1 as a preferred scripting language.
While Lua's syntax is lightweight and easy to learn, it doesn't shy away from convenience features like `func 'arg'` and `func {a=42}`.

LuaJIT offers exceptional performance.

> LuaJIT with the JIT enabled is much faster than all of the other languages benchmarked, including Wren, because Mike Pall is a robot from the future. -- wren.io

This translates into noticeably smoother editing with LSP, especially for hefty C++ files – a significant advantage over Emacs.
With Emacs, I've always felt that editing a large C++ file is slow.

The non-default `local` variables and 1-based indexing (shared with languages like Awk and Julia) are annoyances that I can live with when using a configuration language.
So far, I've only needed index-sensitive looping in one specific location.

```lua
-- For LSP semantic tokens
for type, colors in pairs(all_colors) do
  for i = 1,#colors do
    vim.api.nvim_set_hl(0, string.format('@lsp.typemod.%s.id%s.cpp', type, i-1), {fg=colors[i]})
  end
end
```

## Dual-role keys

I utilize the software keyboard remapper [kanata](https://github.com/jtroo/kanata) to make some keys both as normals keys and as a modifier.
I have followed the guide <https://shom.dev/start/using-kanata-to-remap-any-keyboard/> as the official configuration guide is intimidating.

[`~/.config/kanatta/config.kbd`](https://github.com/MaskRay/Config/blob/master/home/.config/kanata/config.kbd) is my current configuration.
A simplified version is provided below:

```
(defcfg
  concurrent-tap-hold yes
  log-layer-changes no
  process-unmapped-keys yes
)
(defvar
  tt 200 ;; tap-time
  ht 160 ;; hold-time
)

(defalias
  tab (tap-hold $tt $ht tab (layer-while-held extend))
  cap (tap-hold $tt $ht esc lctl)
  ;; cap (tap-hold $tt $hold-time esc (layer-while-held vim-nav))
  a (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht a lmet) break)
  s (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht s lalt) break)
  d (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht d lctl) break)
  f (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht f lsft) break)
  j (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold-release-timeout $tt $ht j rsft j) break)
  k (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold-release-timeout $tt $ht k rctl k) break)
  l (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht l ralt) break)
  ; (switch ((key-timing 1 less-than $tt)) _ break () (tap-hold $tt $ht ; rmet) break)
)

(defsrc
  tab  q    w    e    r    t         y    u    i    o    p    [
  caps a    s    d    f    g         h    j    k    l    ;    '
  lsft z    x    c    v    b         n    m    ,    .    /    rsft
)
(deflayer default
  @tab _    _    _    _    _         _    _    _    _    _    _
  @cap @a   @s   @d   @f   _         _    @j   @k   @l   @;   _
  _    _    _    _    _    _         _    _    _    _    _    _
)
(deflayer extend
  _    _    _    _    lrld _         _    C-S-tab C-tab  _    _    _
  _    _    _    _    _    _         left down up   rght _    _
  _    _    _    _    _    _         home pgdn pgup end  _    _
)

(defchordsv2
  (j  k     ) esc 100 all-released ()
  (   k  l  ) =   100 all-released ()
  (j     l  ) S-= 100 all-released ()
  (      l ;) -   100 all-released ()
)
```
