---
title: Advanced Vim Programming
date: 2020-9-16
tags: vim, advanced
---

## \<SID\>

> The string "<SID>" can be used in a mapping or menu.
>
> When executing the map command, Vim will replace "<SID>" with the special key code <SNR>, followed by a number that's unique for the script, and an underscore.  Example: `:map <SID>Add` could define a mapping "<SNR>23\_Add".

When you map some thing to a `s:` function, which means you may call it may mapping outside of the script. Try the follow scirpts:

``` vimscript
function! s:say()
    echo expand("<SID>")
    echo "hello"
endfunction
map WW :call s:say()<CR>
map RR :call <SID>say()<CR>
```

When mapping `WW` is executed from outside of scirpt, it don't know which the script the function was defined.

The `:scriptnames` command can be used to see which scripts have been sourced


### \<sfile\>

`:h <sfile>`


## \<Plug\>

> The special key name "<Plug>" can be used for an internal mapping, which is not to be matched with any key sequence.  This is useful in plugins

As the doc file said, `<Plug>` key is often use in a internal mapping. Like:

``` vimscript
func! s:somecme
    ...
endfunc
noremap <silent> <Plug>SomeCmd :call <SID>somecmd
nmap ss <Plug>SomeCmd
```

- Why Vim has this feature?
    * may be its a way to set alias in vimscript?


