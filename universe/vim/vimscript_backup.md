---
title: vimscript学习
date: 2020-5-1
tags: backup, vim, vimscript
---

Remenber to use `:h <arg>` for help


## Basic

### Echo

- `:echo "hello"`
- `:echom "hello"`

区别在于echom会将消息放入消息队列中`:messages`


### Setting option

- `:set <name>=<value>`
    * `:set <name>?`可以查看设置的情况


### Abbreviations

- `iabbrev adn and`

在输入模式下输入adn，然后空格会发现变成了and


### Buffer-Local Options

- local mapping
    * like `:map <buffer> D dd`, the `<buffer>`told vim only consider that mapping when we are in the buffer where we dedine it
- local leader
    * `<localleader>` instead of `<leader>`
- setting
    * `setlocal <name>`
- shadowing
    ``` 
    :nnoremap <buffer> Q x
    :nnoremap          Q dd
    ```
    * first mapping is more specific, so the second not work


### Autocommands

``` vim
:autocmd BufNewFile * :write
         ^          ^ ^
         |          | |
         |          | The command to run.
         |          |
         |          A "patterns" to filter the event.
         |
         The "events" to watch for.
```

`:help autocmd-events`


#### Autocommands Group

It is possible to create duplicate autocommands, so if you try running the following command:`:autocmd BufWrite * :sleep 200m` three times. Vim will sleep 600m.

Use `augroup` is the same

- Clearing Groups
    * `autocmd!`

``` vim
:augroup testgroup
:    autocmd!
:    autocmd BufWrite * :echom "Cats"
:augroup END
we enter the testgroup group, immediately clear it(gourp), and no more dup autocommands before any more
```

`:help autocmd-groups`


### Operator-Pending Mapping

- like `:onoremap p i(`, when we run `dp` it was like saying "delete parameters", which vim translates to "delete inside parentheses"


### Status Lines

- `:set statusline=%f\ -\ FileType:\ %y`
- `:set statusline+=`

``` 
:set statusline=%l    " Current line
:set statusline+=/    " Separator
:set statusline+=%L   " Total lines
```

- width and padding
    * `:set statusline=Current:\ %4l\ Total:\ %4L`, like this
        + `Current:   12 Total:  223`
    * `:set statusline=Current:\ %-4l\ Total:\ %-4L`
        + `Current: 12   Total: 223`


## Programming Language

- set up folding for Vimscript files: `set foldmethod=<value>`

Add lines before and after that autocommands group so that it look like thik:

``` vim
" Vimscript file settings ---------------------- {{{
augroup filetype_vim
    autocmd!
    autocmd FileType vim setlocal foldmethod=marker
augroup END
" }}}
```

In normal mode, type `za`. Vim will fold/unfold the lines starting at the one containting `{{{` and ending at `}}}`

`:h foldlevelstart`


### Multiple-Line Statements

like

``` vim
:augroup testgroup
:    autocmd BufWrite * :echom "Baz"
:augroup END
```

You can write this as three separate lines in a Vimscript file. Or you can separate each line with a pipe character `|`. Like this.

``` vim
:echom "foo" | echom "bar"
```


### Variables

- `let` to init or assignment a variable(call it var below)
- `unlet` to delete a var
- `unlet` to delete a var and ignore warnning, if var did not exist

You can add a prefix before :var to define its scope, like this


#### Options as Variables

``` vim
:set textwidth=80
:echo &textwidth
```

Using an ampersand in front of a name tells Vim that you're referring to the option, not a variable that happens to have the same name


#### Registers as Variables

``` vim
:let @a = "hello!"
```

- `:echo @"`, unnamed register, which is where text you yank
- `:echo @/`, echo search pattern you just use


#### Variables Scoping

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

`:h internal-variables`


### Conditionals

Any decision making you do in your coding will be done with `if`s.


#### Basic If

``` vim
:if 1
:    echom "One"
:elseif "nope"
:    echom "elseif"
:else
:    echom "finally"
:endif
```

Vim dose not necessarily treat a non-empty string as "truthly".

try this

``` vim
:echom "hello" + 10
:echom "10hello" + 10
:echom "hello10" + 10
```

- Vim will try to coerce variables (and literals) when necessary. When 10 + "20foo" is evaluated Vim will convert "20foo" to an integer (which results in 20) and then add it to 10.
- Strings that start with a number are coerced to that number, otherwise they're coerced to 0.
- Vim will execute the body of an if statement when its condition evaluates to a non-zero integer, after all coercion takes place.


#### Comparisons

Vim is case senitive so `"foo" != "FOO"`. But if you `:set ignorecase`, vim is case insenitive.

So to be code defensively you can nevertrust the `==` comparison. A bare `==` should never appear in you plugins' code. Vim has two extra sets of comparison operators to deal with this.

- `==?` is the "case-insenitive", no matter what user has set
- `==#` is the "case-senitive", no matter what user has set


#### String comparison

- `<string> == <string>` equal
- `<string> != <string>` not equal
- `<string> =~ <pattern>` matching pattern
- `<string> !~ <pattern>` not matching pattern
- `<operator>#` matching with case
- `<operator>?` matching with ignorecase

`<string>.<string>` to connect string


### Function

 **Vimscript functions must start with a capital letter if they are unscoped!** 

Use `function` keywork to define a function, use `function!` to overwire a function. Be attention, function need to begin with capital letter.

- `:echo Function()` if Function has no return, vim will implicit return 0
- `:delfunction <function>` to delete a function
- `:call <function>` to call a function 


#### Function Arguments

``` vim
function DisplayName(name)
  echom "Hello!  My name is:"
  echom a:name
endfunction
```

Notice the `a:` in the name of the variable represents a variable scope. And it tell vim that they're in the argument scope.


#### Varargs

Vimscript call take variable-length argument lists.

``` vim
function Varg(...)
  echom a:0
  echom a:1
  echo a:000
endfunction

call Varg("a", "b")
```

- `a:0` will be set to the number of extra arguments you were given
- `a:1` will be set to the first extra argument
- `a:000` will be set to a list containing all the extra arguments


#### Assignment

Vimscript not allow you to reassign argument variables.Try running the follow commands, vim will throw an error. And you need a temp var.

``` vim
:function Assign(foo)
:  let a:foo = "Nope"
:  echom a:foo
:endfunction

:call Assign("test")
```


### String

Vimscritp use `.` to connect strings. Use `\` to escape sequences.

Using single quotes tell vim that you want string exactly as-is, with no escape sequences. And two single quotes in a row will produce one single quote. `:echom 'That''s enough.'`


#### String Functions

- Length
    * `strlen("foo")`
    * `len("foo")`
- Splitting: splits a String into a List
    * `split("String", separator)`, separator is "whitespace" by default
- Joining: join a List into a String
    * `join(List, connector)`
- Case
    * `tolower("String")`
    * `toupper("String")`


### Execute

The `execute` command is user to evaluate a string as if it were a Vimscripte command.

#### Normal

Try run

``` vim
:nnoremap G dd
:normal G
```

Vim will delete the current line. The `normal` command will take into account any mapping that exist.

Vim has a `normal!` command to avoid user mapping. So write Vimscript should  **always** use `normal!`.

The problem is that `normal!` dosen't parse special charater sequences like `<cr>`. Combining `normal!` with `execute` fixes that problem. Like this:

``` vim
:execute "normal! gg/print\<cr>"
```






