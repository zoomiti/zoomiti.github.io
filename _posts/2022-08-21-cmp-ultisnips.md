---
title: "Setting up Ultisnips for nvim-cmp"
header:
#  image: /assets/images/2022-08-09-colorscheme.png
tags:
  - Linux
  - neovim
---

Although this blog post may seem unnecessary, the [Ultisnips cmp
source](https://github.com/quangnguyen30192/cmp-nvim-ultisnips) has outdated
documentation. The current issue is that cmp-nvim-ultisnips has not been
updated to properly check if the cmp completion window is actually open. This
means it completely ignores the window and jumps/expands snippets when you
expect to be able to use the completion window.

How do we get around this? In essence by going around cmp-nvim-ultisnips auto
checking and checking manually.

## My cmp Configuration

```lua
local cmp = require'cmp'
local cmp_ultisnips_mappings = require'cmp_nvim_ultisnips.mappings'

cmp.setup({
  snippet = {
    expand = function(args)
      vim.fn["UltiSnips#Anon"](args.body) 
    end,
  },
  mapping = {
    ['<C-d>'] = cmp.mapping.scroll_docs(-4),
    ['<C-f>'] = cmp.mapping.scroll_docs(4),
    ['<C-Space>'] = cmp.mapping.complete(),
    ['<C-e>'] = cmp.mapping.close(),
    ['<CR>'] = cmp.mapping.confirm({ select = true }),
    ['<tab>'] = cmp.mapping(function(fallback)
    if cmp.visible() then
      cmp.select_next_item()
    else
      cmp_ultisnips_mappings.expand_or_jump_forwards(fallback)
    end
    end, { 'i', 's' }),
    ['<S-tab>'] = cmp.mapping(function(fallback)
    if cmp.visible() then
      cmp.select_prev_item()
    else
      cmp_ultisnips_mappings.jump_backwards(fallback)
    end
    end, { 'i', 's' }),
  },
  sources = cmp.config.sources({
    { name = 'nvim_lsp' },
    { name = 'nvim_lua' },
    { name = 'ultisnips' },
    { name = 'path' },
  }, {
    { name = 'buffer', keyword_length = 5},
  }),
})

```
and don't forget to install the necessary plugins to your init.vim:

```vimscript
call plug#begin()

Plug 'hrsh7th/nvim-cmp'  " For LSP completion
Plug 'hrsh7th/cmp-nvim-lsp'
Plug 'hrsh7th/cmp-path'
Plug 'hrsh7th/cmp-buffer'
Plug 'hrsh7th/cmp-nvim-lua'
Plug 'quangnguyen30192/cmp-nvim-ultisnips'

call plug#end()
```
