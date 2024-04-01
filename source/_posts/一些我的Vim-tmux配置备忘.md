---
title: 一些我的 Vim&tmux 配置备忘
date: 2024-04-01 11:16:44
tags: vim, tmux, config
categories: note
---

*注：由于配置又杂，又有很多的来源早找不到了。如果你发现有来自于你的配置但是没有提到，在下面回复一下让我加上就好，阿里嘎多！*

# 前言

有笨蛋在每次准备继续配置 Vim 和 tmux 时，打开自己几个月之前刚该国的配置都会一脸懵……于是在此做个备忘。这并不是一个教程，嗯。

以及首先需要承认我是个 VSC 人，所以以下的 Vim 配置更多的是在 VSC 没法用时做一个替代，并没有完全的美化和配置舒服。但是目前够用了，大概？qw

一个好处是，要是遇上点什么没法装 VSC 的服务器或者小板子，只要有个 ssh 连接，就一个 sftp 丢上去这一套就行 qwq

## Vim

先来个展示：![](https://cdn.jsdelivr.net/gh/wychlw/img@main///img/202404011138220.png)

（如果你发现配置文件里有什么有的但我又没提到的，那说明……我也忘了（扶额）

### 基础设置

```
set nocompatible
filetype on
filetype indent on
filetype plugin on
syntax on

set encoding=utf-8
set spell
set number
set cursorline
set tabstop=4

set incsearch
set ignorecase
set smartcase
set hlsearch

set ruler

"以上emmm 就是这样"

let mapleader="\<space>"
nnoremap <leader>sv :source $MYVIMRC<CR>
```

对于全局的 leader，我使用了 `"\<space>"` （若不正常试试 `" "` 或 `"<space>"`）。最显然的原因是，它很好按，**并且没有被什么奇奇怪怪的东西占用掉**。
由于改配置时肯定要不断测试效果，快捷键 `<leader>sv` 可以快速的重新加载配置。按一下空格再同时按下 `sv` 就行，很方便 ww

### 插件

目前启用了以下插件：
- lightline 状态栏
- nerdtree 文件树
- coc.nvim 代码补全&提示
- LeaderF 文件查找
- catppuccin/nvim 主题

#### lightline

```
let g:lightline = {
                                                \'colorscheme': 'PaperColor'
                                \}
```

采用了 `PaperColor` 主题，主要是只有这个配色看得很清

#### nerdtree

```
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * NERDTree | if argc() > 0 || exists("s:std_in") | wincmd p | endif

autocmd BufEnter * if tabpagenr('$') == 1 && winnr('$') == 1 && exists('b:NERDTree') && b:NERDTree.isTabTree() | quit | endif

autocmd BufEnter * if bufname('#') =~ 'NERD_tree_\d\+' && bufname('%') !~ 'NERD_tree_\d\+' && winnr('$') > 1 |
    \ let buf=bufnr() | buffer# | execute "normal! \<C-W>w" | execute 'buffer'.buf | endif

let g:NERDTreeDirArrowExpandable = '▸'
let g:NERDTreeDirArrowCollapsible = '▾'
```

配置就是官方的，包含自动关闭最后一个窗口。并没有做很多的美化。

#### LeaderF

```
" don't show the help in normal mode
let g:Lf_HideHelp = 1
let g:Lf_UseCache = 0
let g:Lf_UseVersionControlTool = 0
let g:Lf_IgnoreCurrentBufferName = 1
" popup mode
let g:Lf_WindowPosition = 'popup'
let g:Lf_StlSeparator = { 'left': "\ue0b0", 'right': "\ue0b2", 'font': "DejaVu Sans Mono for Powerline" }
let g:Lf_PreviewResult = {'Function': 0, 'BufTag': 0 }

let g:Lf_ShortcutF = "<C-f>"
let g:Lf_DefaultExternalTool='rg'

let g:Lf_WorkingDirectoryMode = 'AF'
let g:Lf_RootMarkers = ['.git', '.svn', '.hg', '.project', '.root']

nmap <leader>fr <Plug>LeaderfRgPrompt
nmap <leader>fra <Plug>LeaderfRgCwordLiteralNoBoundary
nmap <leader>frb <Plug>LeaderfRgCwordLiteralBoundary
nmap <leader>frc <Plug>LeaderfRgCwordRegexNoBoundary
nmap <leader>frd <Plug>LeaderfRgCwordRegexBoundary
vmap <leader>fra <Plug>LeaderfRgVisualLiteralNoBoundary
vmap <leader>frb <Plug>LeaderfRgVisualLiteralBoundary
vmap <leader>frc <Plug>LeaderfRgVisualRegexNoBoundary
vmap <leader>frd <Plug>LeaderfRgVisualRegexBoundary
```

发现自己其实很少使用默认的那堆，所有并没有加上。在我的环境里 `C-f` 并没有被占用，所以干脆就用它来做大窗口的触发了！

然后就是 `<leader>frx` 的直接 `rg` 搜索……基本上我就会用到这些因为。

#### coc.nvim

```
set signcolumn=yes
inoremap <silent><expr> <Tab>
      \ coc#pum#visible() ? coc#pum#next(1) :
      \ CheckBackspace() ? "\<Tab>" :
      \ coc#refresh()
inoremap <expr><S-TAB> coc#pum#visible() ? coc#pum#prev(1) : "\<C-h>"

" Make <CR> to accept selected completion item or notify coc.nvim to format
" <C-g>u breaks current undo, please make your own choice
inoremap <silent><expr> <CR> coc#pum#visible() ? coc#pum#confirm()
                              \: "\<C-g>u\<CR>\<c-r>=coc#on_enter()\<CR>"

function! CheckBackspace() abort
  let col = col('.') - 1
  return !col || getline('.')[col - 1]  =~# '\s'
endfunction

autocmd CursorHold * silent call CocActionAsync('highlight')
xmap <C-I> <Plug>(coc-format)
xmap <C><S>I <Plug>(coc-format)
nmap <C-I> <Plug>(coc-format)
nmap <C><S>I <Plug>(coc-format)

let g:coc_global_extensions = [ 
                                                \ 'coc-json', 
                                                \ 'coc-clangd',
                                                \ 'coc-git'
                                                \]


nmap <leader>c :CocCommand<space>
```

用 `coc.nvim` 来做代码补全。同样截取官方配置。

对于格式化整个文档，部分沿用了 VSC 的默认键位：`C-i`。

#### catppuccin/nvim

```
colorscheme catppuccin-mocha
```

一行启用 `mocha` 主题。看得清就完事了。

### 快捷键

很多系统上， `C-s` `C-q` 等很可能是被占用了的……但是我的没有所有能这么用。

```
nnoremap <A-left> <C-w>h
nnoremap <A-right> <C-w>l
nnoremap <A-up> <C-w>k
nnoremap <A-down> <C-w>j

nnoremap <S-up> <C-w>s
nnoremap <S-down> <C-w>s<C-w>j
nnoremap <S-left> <C-w>v
nnoremap <S-right> <C-w>v<C-w>l
nnoremap <S-h> :sp<space>
nnoremap <S-v> :svp<space>

nnoremap <leader>oh :sp<space>
nnoremap <leader>ov :svp<space>

nnoremap <C-q> <C-w>q
nnoremap <C-s> :w<CR>
inoremap <C-s> <ESC>:w<CR>i
inoremap <C-q> <ESC>:w<CR><C-w>q
```

其中，把切窗口变成了 `Alt+方向键`；
分割窗口变成了 `Shift+方向键` （自动切到新窗口），`S+h/v` 也可以按行/列分割（虽然并用不到？）；
`<leader>ox` 可以按行/列打开新文件。

在命令下使用 `C-s` 保存， `C-q` 退出（不带自动保存）
在插入下 `C-s` 保存并继续编辑，`C-q` 退出**带自动保存**

### 总配置

```
set nocompatible
filetype on
filetype indent on
filetype plugin on
syntax on

set encoding=utf-8
set spell
set number
set cursorline
set tabstop=4

set incsearch
set ignorecase
set smartcase
set hlsearch

set ruler

call plug#begin('~/.vim/plugged')

Plug 'itchyny/lightline.vim'
Plug 'scrooloose/syntastic'
Plug 'scrooloose/nerdtree'
Plug 'neoclide/coc.nvim', {'branch': 'release'}
Plug 'Yggdroot/LeaderF', { 'do': ':LeaderfInstallCExtension' }
Plug 'catppuccin/nvim', { 'as': 'catppuccin' }

call plug#end()

colorscheme catppuccin-mocha

let g:syntastic_c_checkers=['clang_tidy']
let g:syntastic_cpp_checkers=['clang_tidy']
let g:syntastic_always_populate_loc_list = 1
let g:syntastic_auto_loc_list = 1
let g:syntastic_check_on_open = 1
let g:syntastic_check_on_wq = 0


let mapleader="\<space>"
nnoremap <leader>ev :vsp $MYVIMRC<CR>
nnoremap <leader>sv :source $MYVIMRC<CR>

"Begin of LeaderF"

" don't show the help in normal mode
let g:Lf_HideHelp = 1
let g:Lf_UseCache = 0
let g:Lf_UseVersionControlTool = 0
let g:Lf_IgnoreCurrentBufferName = 1
" popup mode
let g:Lf_WindowPosition = 'popup'
let g:Lf_StlSeparator = { 'left': "\ue0b0", 'right': "\ue0b2", 'font': "DejaVu Sans Mono for Powerline" }
let g:Lf_PreviewResult = {'Function': 0, 'BufTag': 0 }

let g:Lf_ShortcutF = "<C-f>"
let g:Lf_DefaultExternalTool='rg'

let g:Lf_WorkingDirectoryMode = 'AF'
let g:Lf_RootMarkers = ['.git', '.svn', '.hg', '.project', '.root']

nmap <leader>fr <Plug>LeaderfRgPrompt
nmap <leader>fra <Plug>LeaderfRgCwordLiteralNoBoundary
nmap <leader>frb <Plug>LeaderfRgCwordLiteralBoundary
nmap <leader>frc <Plug>LeaderfRgCwordRegexNoBoundary
nmap <leader>frd <Plug>LeaderfRgCwordRegexBoundary
vmap <leader>fra <Plug>LeaderfRgVisualLiteralNoBoundary
vmap <leader>frb <Plug>LeaderfRgVisualLiteralBoundary
vmap <leader>frc <Plug>LeaderfRgVisualRegexNoBoundary
vmap <leader>frd <Plug>LeaderfRgVisualRegexBoundary


"end of LeaderF"

"Begin of coc-nvim config"

set signcolumn=yes
inoremap <silent><expr> <Tab>
      \ coc#pum#visible() ? coc#pum#next(1) :
      \ CheckBackspace() ? "\<Tab>" :
      \ coc#refresh()
inoremap <expr><S-TAB> coc#pum#visible() ? coc#pum#prev(1) : "\<C-h>"

" Make <CR> to accept selected completion item or notify coc.nvim to format
" <C-g>u breaks current undo, please make your own choice
inoremap <silent><expr> <CR> coc#pum#visible() ? coc#pum#confirm()
                              \: "\<C-g>u\<CR>\<c-r>=coc#on_enter()\<CR>"

function! CheckBackspace() abort
  let col = col('.') - 1
  return !col || getline('.')[col - 1]  =~# '\s'
endfunction

autocmd CursorHold * silent call CocActionAsync('highlight')
xmap <C-I> <Plug>(coc-format)
xmap <C><S>I <Plug>(coc-format)
nmap <C-I> <Plug>(coc-format)
nmap <C><S>I <Plug>(coc-format)

let g:coc_global_extensions = [ 
                                                \ 'coc-json', 
                                                \ 'coc-clangd',
                                                \ 'coc-git'
                                                \]


nmap <leader>c :CocCommand<space>

"end of coc-nvim config"

"Begin of nerd tree"

autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * NERDTree | if argc() > 0 || exists("s:std_in") | wincmd p | endif

autocmd BufEnter * if tabpagenr('$') == 1 && winnr('$') == 1 && exists('b:NERDTree') && b:NERDTree.isTabTree() | quit | endif

autocmd BufEnter * if bufname('#') =~ 'NERD_tree_\d\+' && bufname('%') !~ 'NERD_tree_\d\+' && winnr('$') > 1 |
    \ let buf=bufnr() | buffer# | execute "normal! \<C-W>w" | execute 'buffer'.buf | endif

let g:NERDTreeDirArrowExpandable = '▸'
let g:NERDTreeDirArrowCollapsible = '▾'

"end of nerd tree"

"Begin fast key for window key config"

nnoremap <A-left> <C-w>h
nnoremap <A-right> <C-w>l
nnoremap <A-up> <C-w>k
nnoremap <A-down> <C-w>j

nnoremap <S-up> <C-w>s
nnoremap <S-down> <C-w>s<C-w>j
nnoremap <S-left> <C-w>v
nnoremap <S-right> <C-w>v<C-w>l
nnoremap <S-h> :sp<space>
nnoremap <S-v> :svp<space>

nnoremap <leader>oh :sp<space>
nnoremap <leader>ov :svp<space>

nnoremap <C-q> <C-w>q
nnoremap <C-s> :w<CR>
inoremap <C-s> <ESC>:w<CR>i
inoremap <C-q> <ESC>:w<CR><C-w>q

"end of window ff-key config"

"Begin of lightline"

let g:lightline = {
                                                \'colorscheme': 'PaperColor'
                                \}

"end of lightline"

```

## Tmux

配置：
```
set-option -g prefix C-x
unbind C-a
bind C-x send-prefix

bind -r Left select-pane -L
bind -r Down select-pane -D 
bind -r Up select-pane -U 
bind -r Right select-pane -R 

bind -r C-w split-window -vb
bind -r C-s split-window -v 
bind -r C-a split-window -hb
bind -r C-d split-window -h

bind -r C-q kill-pane

unbind <
unbind >
bind -r > previous-window 
bind -r < next-window
```

基本常用的窗口命令就是 vim 的需要先 `C-x`。