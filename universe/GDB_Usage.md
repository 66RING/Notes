---
title: GDB usage
date: 2020-2-25
tags: tools, gdb
---

# Basic Usage

- Make sure compile file with `-g` option, which mean turn on debugging with gdb
    - like this: `gcc -g main.cpp -o main`

## Debugging with GDB

- Open file
    - here are two way
    - `gdb <file>`
        - `gdb path/to/main`
    - `gdb` -> `file path/to/main`
- Show list
    - use `l` command to show source code with line number
- Add breakpoint
    - use `b <LINENUM>` to add breakpoint, such as `b 1` add breakpoint at line 1
    - or use `b <FUNCTIONNAME>` to add breakpoint, such as `b main` add breakpoint at main function
- Show breakpoint information
    - `i b` i for information, b for breakpoint
- **Inspect**
    * use `print, inspect or p [OPTION...] [EXP]` to print value of expression EXP
        + use `p a` will show information of variable `a` 
- Delete breakpoint
    - `d <BREAKPOINTNUM>` break point num from `i b`
    - `i b` information will be like a order list. If delete a pointid in the previous, the next point will just add in the end of the idlist.
- Run debug
    - `r`
- Gdb controler
    - `n` **for next line**, one step debugging
    - `s` **for step into**(one step)
    - `set args [parameter]` for command line args
- Finish function and continue
    - `finish` for finish function
    - `c` for continue until program exit or breakpoint
- Quit gdb
    - `q`
