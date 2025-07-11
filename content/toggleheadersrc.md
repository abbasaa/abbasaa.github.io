
---
title: Toggling the header/src file
draft: false
tags:
  - vim
---
 I frequently like switching between my header and cpp file while editing code. It's surprising to me this isn't built in, but I guess writing the function is p easy. This isn't perfect. It first assumes no directory structure and searches in the same directory, otherwise it tries swapping from private to public or vice versa.
 
```lisp
function! ToggleHeaderSource()
  let l:filename = expand('%:p')
  let l:basename = expand('%:r')  " full path without extension
  let l:ext = expand('%:e')       " extension only

  if l:ext ==# 'cpp'
    " First, try same directory .h
    let l:header = l:basename . '.h'
    if filereadable(l:header)
      execute 'edit' l:header
      return
    endif
    " If not found, try swapping Private -> Public
    if l:filename =~ '/Private/'
      let l:header = substitute(l:filename, '/Private/', '/Public/', '')
      let l:header = substitute(l:header, '\.cpp$', '.h', '')
      if filereadable(l:header)
        execute 'edit' l:header
        return
      endif
    endif
    echo "Header file not found."
  elseif l:ext ==# 'h'
    " First, try same directory .cpp
    let l:source = l:basename . '.cpp'
    if filereadable(l:source)
      execute 'edit' l:source
      return
    endif
    " If not found, try swapping Public -> Private
    if l:filename =~ '/Public/'
      let l:source = substitute(l:filename, '/Public/', '/Private/', '')
      let l:source = substitute(l:source, '\.h$', '.cpp', '')
      if filereadable(l:source)
        execute 'edit' l:source
        return
      endif
    endif
    echo "Source file not found."
  else
    echo "Not a .cpp or .h file."
  endif
endfunction
```

