---
title: "How I created my own vim color scheme"
header:
  image: /assets/images/2022-08-09-colorscheme.png
tags:
  - Linux
  - color
  - vim
  - neovim
---

## Motivation
Why make a color scheme when plenty of good ones already exist? Also doesn't
neovim and vim use your terminal colors normally? Well normally it does, but I
decided I wanted to use the `termguicolors` setting which does not respect your
terminal colors. This should've prompted me to use an already prominent color
scheme but in all my searching I did not find one that matched well with my
Orange/Firewatch inspired DWM setup.

## TLDR:
Take a look at my colorscheme [here](https://github.com/zoomiti/firewatch). It
was built using the [vim-colortemplate](#vim-colortemplate) vim plugin.

## [Vim-Colortemplate](https://github.com/lifepillar/vim-colortemplate)

This tool made the creation of my color scheme ridiculously simple... once I figured it out.

### Initial Hurdles
Although any color schemes created with this tool is compatible with vim and
neovim the plugin only works in vim 8.0 or above.

### Usage
This plugin adds a new file type known as `colortemplate` with, funnily enough,
its own syntax highlighting built in. It supports creating a palette to reuse
colors in every highlight statement converting
```vim
hi Normal guifg=#f2a97d guibg=#381818 gui=NONE cterm=NONE
```
into a much simpler
```
Normal		white		black
```
assuming you defined white and black earlier in the `colortemplate` file like this
```
; Color name	GUI		Base256		Base16 (optional)
Color: white	#ffffff		~		White
Color: black	#000000		~		Black
```

Lastly this format also supports defining metadata including its name and its author 
like:

```{% raw %}
; Information {{{
Full name:  My Gorgeous Theme
Short name: gorgeous
Author:     Me <me@somewhere.org>
; }}}
```
{% endraw %}

All put together a minimal (compilable) `colortemplate` can look like this (taken from the [README](https://github.com/lifepillar/vim-colortemplate#readme) in the GitHub repo).
```
Full name:  My Gorgeous Theme
Short name: gorgeous
Author:     Me <me@somewhere.org>

Variant:    gui 256
Background: dark

; Color palette
Color:      myblack #333333 ~
Color:      mywhite #fafafa ~

; Highlight group definitions
Normal      mywhite myblack

Term colors: mywhite mywhite mywhite mywhite mywhite mywhite mywhite mywhite
Term colors: myblack myblack myblack myblack myblack myblack myblack myblack

```

### Plugin Support
Plugin support varies from trivial to slightly involved. For example supporting
[nvim-cmp](https://github.com/hrsh7th/nvim-cmp) was as easy as adding these
lines to the colortemplate.

```vim
CmpItemMenu -> Comment
CmpItemKindDefault -> Identifier
CmpItemAbbrMatch -> Pmenu
CmpItemKindDefault -> Pmenu
CmpItemKindFunction -> Function
CmpItemKindMethod -> CmpItemKindFunction
CmpItemKindModule -> PreProc
CmpItemKindStruct -> CmpItemKindModule
CmpItemKindText -> Comment
CmpItemKindSnippet -> Constant
CmpItemKindReference -> Identifier
CmpItemKindInterface -> Identifier
```

But for something like [lightline](https://github.com/itchyny/lightline.vim) it involved using some macros to define a new file in the `autoload/lightline/colorscheme` directory like this:

```vim
auxfile autoload/lightline/colorscheme/@shortname.vim
...
endauxfile
```

For some examples of this you can look at [my colorscheme](https://github.com/zoomiti/firewatch/blob/main/templates/_lightline).

