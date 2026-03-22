# vim / neovim command ref

> Most of this applies to both Vim and Neovim. Neovim-specific sections are marked **[nvim]**.

## modes

| Mode | Enter with | Indicator |
|------|-----------|-----------|
| Normal | `Esc` / `Ctrl-[` | (default) |
| Insert | `i` / `a` / `o` | `-- INSERT --` |
| Visual | `v` | `-- VISUAL --` |
| Visual Line | `V` | `-- VISUAL LINE --` |
| Visual Block | `Ctrl-v` | `-- VISUAL BLOCK --` |
| Command | `:` | `:` prompt |
| Replace | `R` | `-- REPLACE --` |

## navigation — normal mode

### basic motion

```
h j k l          left / down / up / right
w W              next word start (W = whitespace-delimited)
b B              prev word start
e E              next word end
0                start of line (col 0)
^                first non-blank char of line
$                end of line
gg               top of file
G                bottom of file
<n>G             go to line n
<n>gg            go to line n
Ctrl-d           scroll half-page down
Ctrl-u           scroll half-page up
Ctrl-f           scroll full page down
Ctrl-b           scroll full page up
zz               center current line in window
zt               current line to top
zb               current line to bottom
```

### jumping

```
%                jump to matching bracket / paren / tag
{  }             jump to prev / next blank line (paragraph)
(  )             jump to prev / next sentence
[[  ]]           jump to prev / next section / function
gd               go to definition (local)
gD               go to definition (global)
gi               go to last insert position and re-enter insert
Ctrl-o           jump back (previous location)
Ctrl-i           jump forward
'.               jump to last edit position
``               jump to last jump position
```

### marks

```
ma               set mark a
`a               jump to mark a (exact position)
'a               jump to mark a (line start)
:marks           list all marks
```

## inserting text

```
i                insert before cursor
I                insert at start of line
a                append after cursor
A                append at end of line
o                open new line below
O                open new line above
ea               append at end of word
Esc / Ctrl-[    return to normal mode
```

## editing — normal mode

### operators (combine with motions)

```
d<motion>        delete (cut)
c<motion>        change (delete + enter insert)
y<motion>        yank (copy)
><motion>        indent right
<<motion>        indent left
=<motion>        auto-indent
```

### common combos

```
dd               delete line
D                delete to end of line
cc               change entire line
C                change to end of line
yy               yank line
Y                yank to end of line (or full line, same as yy in Neovim)
x                delete character under cursor
X                delete character before cursor
s                substitute character (delete + insert)
S                substitute line
r<char>          replace character under cursor
~                toggle case of character
g~<motion>       toggle case of motion
gU<motion>       uppercase motion
gu<motion>       lowercase motion
.                repeat last change
u                undo
Ctrl-r           redo
```

### cut / copy / paste

```
p                paste after cursor / below line
P                paste before cursor / above line
"ay              yank into register a
"ap              paste from register a
"+y              yank to system clipboard
"+p              paste from system clipboard
:reg             show all registers
```

### text objects (use with operators)

```
iw / aw          inner word / a word (with space)
is / as          inner sentence / a sentence
ip / ap          inner paragraph / a paragraph
i" / a"          inner / outer double quotes
i' / a'          inner / outer single quotes
i` / a`          inner / outer backticks
i( / a(          inner / outer parentheses  (also ib)
i[ / a[          inner / outer brackets
i{ / a{          inner / outer braces  (also iB)
it / at          inner / outer HTML tag
```

Examples: `daw` delete a word, `ci"` change inside quotes, `yip` yank paragraph.

## search & replace

```
/pattern         search forward
?pattern         search backward
n / N            next / prev match
*                search for word under cursor (forward)
#                search for word under cursor (backward)
:%s/old/new/g    replace all in file
:%s/old/new/gc   replace all, confirm each
:s/old/new/g     replace in current line
:5,10s/old/new/g replace in lines 5-10
:noh             clear search highlight
```

### search flags

```
/pattern\c       case-insensitive
/pattern\C       case-sensitive
/\<word\>        match whole word only
```

## command mode

```
:w               save
:w <file>        save as
:q               quit
:q!              quit without saving
:wq / :x / ZZ   save and quit
:e <file>        open file
:e!              reload file from disk
:r <file>        insert file contents below cursor
:!<cmd>          run shell command
:r !<cmd>        insert shell command output
:set <option>    set option
:set no<option>  unset option
:set <option>?   check option value
:verbose set <option>?  show where option was set
```

## windows & splits

```
:sp <file>       horizontal split
:vsp <file>      vertical split
Ctrl-w s         horizontal split
Ctrl-w v         vertical split
Ctrl-w w         cycle windows
Ctrl-w h/j/k/l   move to window left/down/up/right
Ctrl-w H/J/K/L   move window to far left/bottom/top/right
Ctrl-w =         equalize window sizes
Ctrl-w +/-       increase / decrease height
Ctrl-w >/<       increase / decrease width
Ctrl-w _         maximize height
Ctrl-w |         maximize width
:q               close current window
:only            close all other windows
```

## tabs

```
:tabnew <file>   open in new tab
:tabn / :tabp    next / prev tab
gt / gT          next / prev tab (normal mode)
<n>gt            go to tab n
:tabclose        close tab
:tabonly         close all other tabs
:tabs            list all tabs
```

## buffers

```
:ls / :buffers   list open buffers
:b <n>           switch to buffer n
:b <name>        switch to buffer by name (tab-complete)
:bn / :bp        next / prev buffer
:bd              close buffer
:bd!             force close buffer
:bufdo <cmd>     run command on all buffers
```

## visual mode

```
v                character-wise visual
V                line-wise visual
Ctrl-v           block-wise visual
o                move to other end of selection
O                move to other corner (block mode)
gv               re-select last visual selection
```

In visual mode, operators work on the selection: `d` delete, `y` yank, `c` change, `>` indent, `=` re-indent, `:` command on range.

## macros

```
qa               record macro into register a
q                stop recording
@a               play macro a
@@               replay last macro
5@a              play macro a 5 times
:norm @a         run macro on every line in range
```

## folding

```
zf<motion>       create fold
zo / zO          open fold / open all nested
zc / zC          close fold / close all nested
za               toggle fold
zR               open all folds
zM               close all folds
zd               delete fold
```

## useful settings (vimrc / init.vim)

```vim
set number          " line numbers
set relativenumber  " relative line numbers
set expandtab       " spaces instead of tabs
set tabstop=2       " tab width
set shiftwidth=2    " indent width
set smartindent     " auto-indent
set wrap            " line wrap
set ignorecase      " case-insensitive search
set smartcase       " case-sensitive if uppercase used
set incsearch       " incremental search
set hlsearch        " highlight matches
set clipboard=unnamedplus  " use system clipboard
set scrolloff=8     " keep 8 lines above/below cursor
set signcolumn=yes  " always show sign column
set updatetime=250  " faster diagnostics
```

---

## neovim-specific [nvim]

### config location

```
~/.config/nvim/init.lua        # Lua config entry point
~/.config/nvim/init.vim        # Vimscript entry point (either/or)
~/.config/nvim/lua/            # Lua module directory
```

### plugin managers

**lazy.nvim** (recommended — modern, lazy-loading, lock file):

```lua
-- bootstrap lazy.nvim in init.lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({ "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git", lazypath })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup("plugins")  -- loads lua/plugins/*.lua
```

```bash
:Lazy           # open plugin manager UI
:Lazy sync      # install + update + clean
:Lazy update    # update plugins
:Lazy clean     # remove unused plugins
:Lazy profile   # startup profiling
```

### popular plugins (lazy.nvim spec format)

```lua
-- lua/plugins/init.lua (examples)
return {
  -- fuzzy finder
  { "nvim-telescope/telescope.nvim", dependencies = { "nvim-lua/plenary.nvim" } },

  -- file tree
  { "nvim-tree/nvim-tree.lua" },

  -- syntax highlighting
  { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate" },

  -- LSP
  { "neovim/nvim-lspconfig" },
  { "williamboman/mason.nvim" },           -- LSP/linter installer
  { "williamboman/mason-lspconfig.nvim" },

  -- completion
  { "hrsh7th/nvim-cmp" },
  { "hrsh7th/cmp-nvim-lsp" },

  -- git signs in gutter
  { "lewis6991/gitsigns.nvim" },

  -- statusline
  { "nvim-lualine/lualine.nvim" },

  -- colorscheme
  { "folke/tokyonight.nvim" },

  -- autopairs
  { "windwp/nvim-autopairs" },

  -- commenting
  { "numToStr/Comment.nvim" },

  -- which-key (shows keybind hints)
  { "folke/which-key.nvim" },
}
```

### LSP basics [nvim]

```
gd               go to definition
gD               go to declaration
gr               go to references
gi               go to implementation
K                hover documentation
<leader>rn       rename symbol
<leader>ca       code action
[d / ]d          prev / next diagnostic
:LspInfo         show active LSP servers
:Mason           open Mason UI
```

### telescope [nvim]

```
:Telescope find_files         find files
:Telescope live_grep          grep across project
:Telescope buffers            list open buffers
:Telescope help_tags          search help docs
:Telescope git_commits        browse git log
:Telescope git_status         show git status
```

### treesitter [nvim]

```
:TSInstall <lang>    install parser (e.g. python, lua, bash)
:TSUpdate            update all parsers
:TSBufToggle highlight  toggle highlighting
```

### neovim terminal

```
:terminal        open terminal in buffer
:vsp | terminal  open terminal in vertical split
Ctrl-\ Ctrl-n    exit terminal insert mode (back to normal)
```

### lua keymapping pattern [nvim]

```lua
vim.keymap.set("n", "<leader>ff", ":Telescope find_files<CR>", { desc = "Find files" })
vim.keymap.set("n", "<leader>e",  ":NvimTreeToggle<CR>",       { desc = "File tree" })
vim.keymap.set("v", "<",          "<gv",                        { desc = "Indent left" })
vim.keymap.set("v", ">",          ">gv",                        { desc = "Indent right" })
```

### health check [nvim]

```
:checkhealth          full health report
:checkhealth nvim     core only
:checkhealth lazy     plugin manager
```
