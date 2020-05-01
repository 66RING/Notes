---
title: vimscript学习
date: 2020-5-1
tags: backup, vim, vimscript
---

Remenber to use `:h <arg>` for help

## Variable

- `let` to init or assignment a variable(call it var below)
- `unlet` to delete a var
- `unlet` to delete a var and ignore warnning, if var did not exist

You can add a prefix before :var to define its scope, like this

``` 
g:var  # as global
a:var  # as a function arg
l:var  # as a function var
b:var  # as buffer var
w:var  # as window var
t:var  # as tab var
s:var  # as script var, which useful in this script only
v:var  # as vim build-in var
```

## Data type

- Number
- Float
- String
- Funcref
    - reference of a function
    - `let func = function("strlen")`
- List
- Dictionary


## String comparison

- `<string> == <string>` equal
- `<string> != <string>` not equal
- `<string> =~ <pattern>` matching pattern
- `<string> !~ <pattern>` not matching pattern
- `<operator>#` matching with case
- `<operator>?` matching with ignorecase

`<string>.<string>` to connect string


## Control

`If, For, While, Try/Catch`


## Function

Use `function` keywork to define a function, use `function!` to overwire a function. Be attention, function need to begin with capital letter.

- `delfunction <function>` to delete a function
- `call <function>` to call a function 




