
---
title: Add window numbers to Vim
draft: false
tags:
  - vim
---
 No shot I can write vim lisp on my own, but this is relatively straightforward. When using vim with a lot of tabs and windows, I want to know which window number I'm on. Vim has 2 built in convenience functions: `win_getid`, `win_id2win`.

We can then write a function that just returns a string with the window number. This function can be called by our `vim-airline` plugin.

Add this to your `.vimrc`
```lisp
" Function to get window number
function! WindowNumber()
  return '(' . win_id2win(win_getid()) . ')'
endfunction

" Customize vim-airline section c
let g:airline_section_c = airline#section#create_left([
      \ '%f',
      \ '%{WindowNumber()}'
      \ ])
```

