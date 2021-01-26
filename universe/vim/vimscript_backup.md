---
title: learn vimscript the hard way
date: 2020-5-1
tags: 
- backup
- vim
- vimscript
---

Remenber to use `:h <arg>` for help

Ref: [https://learnvimscriptthehardway.stevelosh.com/](https://learnvimscriptthehardway.stevelosh.com/)

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

In normal mode, type `za`. Vim will fold/unfold the lines starting at the one containting `3{` and ending at `3}`

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

The problem is that `normal!` dosen't parse special character sequences like `<cr>`. Combining `normal!` with `execute` fixes that problem. Like this:

``` vim
:execute "normal! gg/print\<cr>"
```


### Demos

#### Grep Operation, Part One

`:nnoremap <leader>g :grep -R '<cWORD>' %<cr>`


##### Escaping Shell Command Argument

Try the mapping on the word `that's`. It won't work, because the single quote inside the word interferes with the quotes in the grep command!

To get around this we can use Vim's `shellescape` function. `:h shellescape()` and `:h escape`

The problem is that **Vim performed the `shellescape()` call before it expended out special string like `<cword>`.** So `:echom shellescape("<cWORD>")` is literally output `'<cword>'`

To fix this we'll **use the `expand()` function to force th expansion of `<cword>` into the actual string** before it gets passed to `shellescape`.

try following over the word like `that's`:

``` vim
:echom expand("<cWORD>")
:echom shellescape(expand("<cWORD>"))
```

Finally, try `:nnoremap <leader>g :silent execute "grep! -R " . shellescape(expand("<cWORD>")) . " ."<cr>:copen<cr>`

- `:h copen`, 
    * Open a window to show the current list of errors
- `:h silent`
    * The `silent` command just runs the command that follows it while hiding any messages it would normally display


#### Grep Operator, Part Two

- Create a File
    * Inside `.vim/plugin` create file named `grep-operato.vim`
    * This is where you'll place the code.
- Skeleton
    * To create a new Vim operator you'll start with two components:a function and a mapping
        ``` vim
        nnoremap <leader>g :set operatorfunc=GrepOperator<cr>g@

        function! GrepOperator(type)
            echom "Test"
        endfunction
        ```
    * we set the `operatorfunc` option to our function
    * then we run `g@` which calls this function as an operator
        + **try type `g@` in normal mode**
- Visual Mode
    * Add another mapping `vnoremap <leader>g :<c-u>call GrepOperator(visualmode())<cr>`
        + `<c-u>` to say "delete from the cursor to the bebginning of the line"
        + **`visualmode()` is a built-in Vim function** that return a one-character string representing the last type of visual mode. "v、V and Crtl-v"
    

##### Motion type

``` vim
nnoremap <leader>g :set operatorfunc=GrepOperator<cr>g@
vnoremap <leader>g :<c-u>call GrepOperator(visualmode())<cr>

function! GrepOperator(type)
    echom a:type
endfunction
```

- Pressing `viw<leader>g` echoes `v` because we were in characterwise visual mode.
- Pressing `Vjj<leader>g` echoes `V` because we were in linewise visual mode.
- Pressing `<leader>giw` echoes `char` because we used a characterwise motion with the operator.
- Pressing `<leader>gG` echoes `line` because we used a linewise motion with the operator.


##### Copying the Text

Edit a function look like this:

``` vim
nnoremap <leader>g :set operatorfunc=GrepOperator<cr>g@
vnoremap <leader>g :<c-u>call GrepOperator(visualmode())<cr>

function! GrepOperator(type)
    if a:type ==# 'v'
        execute "normal! `<v`>y"
    elseif a:type ==# 'char'
        execute "normal! `[v`]y"
    else
        return
    endif

    echom shellescape(@@)
endfunction
```

- `==#` case-sensitive comparison
- **`normal!` dose two things**:
    * Visually select the range of text we want by:
        + Moving to mark at the beginning of the range.
        + Entering characterwise visual mode.
        + Moving to the mark at the end of the range.
    * Yanking the visually selected text.
- `@@` is the "unnamed" register: the one that Vim places text into then yank or delete by default


#### Grep Operator, Part Three

**Namespacing** 

Our script create a function named `GrepOperator` in the global namespace.

We can avoid polluting the global namespace by tweaking a couple of lines in our code.

``` vim
nnoremap <leader>g :set operatorfunc=<SID>GrepOperator<cr>g@
vnoremap <leader>g :<c-u>call <SID>GrepOperator(visualmode())<cr>

function! s:GrepOperator(type)
endfunction
```

- modified the function name to start with `s:` which places it in the current script's namespace
- modified the mapping and prepended the `GrepOperator` function name with `<SID>` so they could find the function.
    * If we hadn't done this they would have tried to find the function in the global namespace, which wouldn't have worked

`:h <SID>`
 

### List

Vim List is much like python.

- Lists
    * `:echo ['foo', 3, 'bar']`
    * `:echo ['foo', [3, 'bar']]` of course can be nested
- Indexing
    * `:echo [0, [1, 2]][1]` will display `[1, 2]`
    * `:echo [0, [1, 2]][-2]` will display `0`
- Slicing
    * `:echo ['a', 'b', 'c', 'd', 'e'][0:2]`
    * `:echo ['a', 'b', 'c', 'd', 'e'][:2]`
- Concatenation
    * `:echo ['a', 'b'] + ['c']`
- Built-in List Function
    * `:call add(list, 'b')` append `b` 
    * `:echo len(list)`
    * `:echo get(foo, 100, 'default')`
        + The `get` function will get the item at the given index from the given list, or return the given default value if the index is out of range in the list.
    * `:echo index(foo, 'b')`
        + return the first index of `b`, or `-1` if not in list
    * `:echo join(list, '--')` join item in list together into a string
    * `:call reverse(list)`

`:h functions`


### Looping

- For Loops
    ``` vim
    for i in [1, 2, 3]
    endfor
    ```
- While Loops
    ``` vim
    let c = 1
    while c <= 4
        let c += 1
    endfor
    ```


### Dictionaries

- Dictionaries
    * `:echo {'a': 1, 100: 'foo'}`
- Indexing
    * `:echo {'a': 1, 100: 'foo',}['a']`
    * Or Javascritp-style "dot" `:echo {'a': 1, 100: 'foo',}.a`
- Assigning and Adding `:let foo = {'a': 1}`
    * Assigning: `:let foo.a = 100` 
    * Add: `:let foo.b = 200`
- Removing Entries
    * `:let test = remove(foo, 'a')`
        + Remove the entry with the given key
        + And return the removed value
    * `unlet foo.b`
        + Also removes entries, but you can't use the value
- Dictionary Functions
    * `:echom get({'a': 100}, 'b', 'default')`
        + just like get function for lists
    * `:echom has_key({'a': 100}, 'a')`
        + return `0` as falsy
    * `:echo items({'a': 100, 'b': 200})`
        + You can pull the key-value pairs out of a dict with `item`
        + display like `[['a', 100], ['b', 200]]`


### Toggling

For boolean options we can use `set someoption!` to "toggle" the option.If we want to toggle a non-boolean option we'll need to do a bit more work.

Like this. Remember Vim treats the 0 as falsy and any other number as truthy

``` vim
function! FoldColumnToggle()
    if &foldcolumn
        setlocal foldcolumn=0
    else
        setlocal foldcolumn=4
    endif
endfunction

" Or

function! QuickfixToggle()
    if g:quickfix_is_open
        cclose
        let g:quickfix_is_open = 0
    else
        copen
        let g:quickfix_is_open = 1
    endif
endfunction
```

- `:h wincmd`
- `:h winnr()`


### Functional Programming


#### Immutable Data Structures

Vim dosen't have any immutable collections, but by create some helper functions we can fake it to some degree.

Like this:

``` vim
function! Sorted(l)
    let new_list = deepcopy(a:l)
    call sort(new_list)
    return new_list
endfunction
```

Vim's `sort()` sorts the list in place, so we create a full copy so original list won't changed.


#### Functions as Variables

Vimscript supports using variables to store functions, but the syntax is a bit obtuse.

If a Vimscript variable refers to a function it must start with a capital letter.


#### Higher-Order Functions

``` vim
function! Mapped(fn, l)
    let new_list = deepcopy(a:l)
    call map(new_list, string(a:fn) . '(v:val)')
    return new_list
endfunction

:h map()
:h function()
```


### Paths

- Relative Paths
    * `:echom expand('%')`. `%`means the file you're editing.
- Absolute Paths
    * `:echom expand('%:p')`. The `:p` in the string tells Vim that you want the absolute path.
    * `:echo fnamemodify('foo.txt', ':p')` did the same. `fnamemodify()` is a Vim function that's more flexible than `expand()` in that you can specify any file name, not just special stirngs.
- Listing Files
    * `:echo split(globpath('.', '*'), '\n')`, list of files in a specific directory(here is all files of current dir).
    * You can recursively list file with `**`:`:echo split(globpath('.', '**'), '\n')`


## Creating a Full Plugin

### Basic Layout

- Files in `~/.vim/colors` are treated as color schemes
- Files in `~/.vim/plugin` will each be run once every time Vim starts
- Files in `~/.vim/ftdetect` will also be run every time you start Vim
    * `ftdetect` stands for "filetype detection"
    * The file in this dir should set up autocommmands that detect and set the `filetype` of files
- Files in `~/.vim/ftplugin`. When Vim sets a buffer's `filetype` to a value it then looks for a file(or dir) in `ftplugin` that matches and run it.
    * The naming of these files matters!
    * Because these files are run every time a buffer's filetype is set they must only set buffer-local options! If they set options globally they would overwrite them for all open buffers!
- Files in `~/.vim/indent` are a lot like `ftplugin` files. They get loaded based on their names.
- Files in `~/.vim/compiler` work exactly like `indent` files.
    * They should set compiler-related options in the current buffer based on their names
- Files in `~/.vim/after` Files in this dir will be loaded every time Vim start, but after the file in `~/.vim/plugin`
- Files in `~/.vim/autoload` 
- Files in `~/.vim/syntax` When Vim sets a buffer's `filetype` to a value it then looks for a file(or dir) in `syntax` that matches and run it.
- Files in `~/.vim/doc` is where you can add doc for your plugin


### Runtimepath

Much like `PATH` on `Linux/Unix/BSD` systems, Vim has the runtimepath setting which tells it where to find files to load.

`:set runtimepath?`

So a plugin manager automatically add paths to your `runtimepath` when you load vim


### Detecting Filetype

Vim dosen't what a `.type` file is yet.

Create `ftdetect/TYPE.vim`

`au BufNewFile,BufRead *.TYPE set filetype=TYPE`


### Basic Syntax Highlighting

- `:help syn-keyword`
- `:help iskeyword`
- `:help group-name`
- `:help syntax`

Create a `syntax/potion.vim` file and set you buffer filetype to potion `:set filetype=potion`. The `to`, `times` and `if` words will be highlighted as keywords.

``` vim
syntax keyword potionKeyword to times
syntax keyword potionKeyword if
highlight link potionKeyword Keyword
```

These two lines show the basic structure of simple sytax highlight in Vim.

- You first define a "chunk" of syntax using `syntax keyword` or a related command (which we'll talk about later)
- You then link "chunks" to highlighting groups. A highlighting group is something you define in a color scheme, for example "function names should be blue".


#### Highlighting Functions

Another standard Vim highlighting group is `Function`.

``` vim
syntax keyword potionFunction print join string
highlight link potionFunction Function
```


#### Advanced Syntax Highlighting

Because some character not in `iskeyword` we need to use a regular expression to match it. We'll do this with `syntax match` instead of `syntax keyword`.

``` vim
syntax match potionComment "\v#.*$"
highlight link potionComment Comment
```

We use `syntax match` which tells Vim to match regexes instead of literal keywords.


#### Highlighting Operators

Another part we need to regexes to highlight is operators. `:h group-name`

``` vim
syntax match potionOperator "\v-\="
highlight link potionOperator Operator
```


#### Highlighting Strings

`:help syn-region`

To highlight strings, we'll use the `syntax region` command.

``` vim
syntax region potionString start=/\v"/ skip=/\v\\./ end=/\v"/
highlight link potionString String
```

You'll see the string between "" is highlighted!

- Regions hava a "start" pattern and an "end" pattern that specify where they start and end
- The `skip` argument tells Vim to ignore this matching region
    * The "skip" argument to `syntax region` allow us to handle string escapes like `"she said: \"hello\"!"`


### Basic Folding

- `:help usr_28`
- `:help fold-XXX`


- Manual
    * You create the folds by hand and they stored in RAM. When you close Vim they go away.
- Marker
    * Vim folds your code based on characters in the actual text.
    * Usually these characters put in comments (like`3{`), but in some languages you can get away with using something in the language's syntax itself, like `{` in javascript file.
    * It may seem ugly to clutter up your code with comments, but it let you hand-craft folds for a specific file.
- Diff
    * A special folding mode used when diff'ing files.
- Expr
    * This lets you use a custom piece of Vimscript to define where folds occur.
- Indent
    * Vim uses your code's indentation to determine folds.

`setlocal foldmethod=<method>` and play around with `zR`, `zM`, `za`, `zf`...


#### Advanced Folding

- Folding Theory
    * Each line of code in a file has a "foldlevel". This is always either zero or a positive interger.
    * Lines with a foldlevel of zero are never include in any fold
    * Adjacant line with the same foldlevel are folded togerther
    * If a fold level X is closed, any subsequent lines with a foldlevel greater then or equal to X are folded along with it until you reach a line with a level less than X.


##### Expr Folding

``` vim
setlocal foldmethod=expr
setlocal foldexpr=GetPotionFold(v:lnum)

function! GetFold(lnum)
    return '0'
endfunction
```

- first line tells Vim to use `expr` folding
- second line defines the expression Vim should use to get the foldlevel of a line
    * When Vim runs the expression it will set `v:lnum` to the line number of the ling it wants to know about.
- The expr function return a String and not an Integer

Our funciont return `'0'` for every lines, so Vim won't fold andthing at all.


##### Blank Lines

Modify the `GetFold` function to look like this:

``` vim
function! GetPotionFold(lnum)
    if getline(a:lnum) =~? '\v^\s*$'
        return '-1'
    endif

    return '0'
endfunction
```

- Use `getline(a:lnum)` to get the content of the current line as a String.
    * This regex `\v^\s*$` will match "beginning of line, any number of whitespace characters, end of line"


##### Special Foldlevels

Your custom folding expression can return one of a few "special" strings that tell Vim how to fold the line.

`-1` is one of these special strings. It tells Vim that the level of this line is "underfined", which means "the foldlevel of this line is equal to the foldlevel of the line above or below it"


##### An Indentation Level Helper

To tackle non-blank lines we'll need to know their indentation level.

``` vim
function! IndentLevel(lnum)
    return indent(a:lnum) / &shiftwidth
endfunction
```

And run `:echom IndentLevel(1)` to see line 1 indent level.


### Section Movement Theory

If you've never used Vim's section movement commands(like: `[[`, `]]`, `[]`, `][`) take a second and read `:help section`


#### Nroff Files

The four "section movement" commands are conceptually meant to move around between "sections" of a file.

These commands are designed to work with [nroff files](https://en.wikipedia.org/wiki/Nroff) by default.


### External Commands

The `:!` command (pronounced "bang") in Vim runs external commands and display their output on the screen.

Vim doesn't pass any input to the command then run this way: `:!cat`

To run an external command without the `Press ENTER or type command to continue` prompt, use `:silent !`. Note that this command is `:silent !` and not `:silent!`


#### system()

The `system()` Vim function takes a command string as a parameter and returns the output of that command as String.

You can pass a second string as an argument to `system()`. Run the following command: `:echom system("wc -c", "abcdefg")`

If you pass a second argument like this, Vim will write it to a temporary file and pipe it into the command on standard input.


#### Scratch Splits

`:split BufferName` create an new splite for a buffer named `BufferName`


#### append()

The `append()` Vim function takes two arguments: a line number to append after, and a list of Strings to append as lines. For example:

``` vim
call append(3, ["foo", "bar"])
```

`:help append()`


#### split()

`split()` takes a String to split and regular expression to find the split points.

`:help split()`


### Autoloading

`:help autoload`

Autoload lets you delay loading code until it's actually needed.

Look at the following command:

``` vim
:call somefile#Hello()
```

If this function has already been loaded, Vim will simply call it normally. Otherwise Vim will look for a file called `autoload/somefile.vim`.

If this file exists, Vim will load/source that file. It will then try to call the function normally.

Inside this file, the function should be defined like this:

``` vim
function somefile#Hello()
    " ...
endfunction
```

You can use multiple `#` characters in the function name like `:call myplugin#somefile#Hello()`. And it will look for `autoload/myplugin/somefile.vim`

If a autoload function like `somefile#Hello()` has been loaded, Vim doesn't need to reload the file to call it. 

If `somefile#A()` has been load and `somefile#B()` doesn't load. At that time, you rewrite function A. Without closing Vim and call funcion B, Vim will reload this file and your modify in function A will be update.


### Documentetion

#### How Documentetion Works

A documentation `filetype=help`.

Create a file called `doc/somefile.txt` in you plugin repo. This is where we'll write help for our plugin.

Open this file in Vim and run `:set filetype=help` so you can see the syntax highlighting as you type.


#### Help Header

The format of help files is a matter of personal taste, but better to be popular with the modern Vimscript community.

- The first line of the file should contain the filename of the help file.
    * `*myhelp.txt* functionality for the potion programming language`
- Surrounding a word witd asterisks in a help file create a "tag" that can be jumped to.
    * Run `:Helptags` to rebuild the index of help tags, and then open a new Vim window and run `:help myhelp.txt`. Vim will open your help document like any other one.
- Title, description, authors and something
    * The `~` characters at the end of the lines ensure that Vim dosen't try to highlight or hide individual characters inside the art.


#### Table of Contents

``` vim
====================================================
CONTENTS                                            *PotionContents*

    1. Usage ................ |PotionUsage|
    2. Mappings ............. |PotionMappings|
    3. License .............. |PotionLicense|
    4. Bugs ................. |PotionBugs|
    5. Contributing ......... |PotionContributing|
    6. Changelog ............ |PotionChangelog|
    7. Credits .............. |PotionCredits|
```

- The line of `=` character will be syntax highlight. Use to divide.
    * You can also use line of `-`
- The `*PotionContents*` will create another tag, so run `:help PotionContents` to go directly to the table of contents.
- Each of the word surrounded by `|` create a ling to a tag
    * Users can press `<c-]>` in the file file to jump to the tag, Or click with mouse


### The Command Command

Read `:help user-commands`

Some plugin hava commands like `:Gdiff` and leave it up to the user to decide how to call them.

Commands like this are create with the `:command` command.


### Omnicomplete

- `:help ins-completion`
- `:help omnifunc`
- `:help compl-omni`

Vim offer a number of different ways to complete text. The most powerful of them is "omnicomplete" which let you call a custom Vimscirpt function to determin completions.


### Compiler Support

Read `:help quickfix.txt`

Vim offer much deeper support for interacting with compilers, including parsing compile error.



