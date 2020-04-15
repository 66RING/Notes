---
title: GDB usage
date: 2020-2-25
tags: tools, gdb
---

# Basic Usage

- Make sure compile file with `-g` option, which mean turn on debugging with gdb
    - like this: `gcc -g main.cpp -o main`

## Debugging with GDB

- open file
    - here are two way
    - `gdb filename`
        - `gdb path/to/main`
    - `gdb` -> `file path/to/main`
- show list
    - use `l` command to show source code with line number
- add breakpoint
    - use `b linenum` to add breakpoint, such as `b 1` add breakpoint at line 1
    - or use `b functionname` to add breakpoint, such as `b main` add breakpoint at main function
- show breakpoint information
    - `i b` i for information, b for breakpoint
- delete breakpoint
    - `d pointid` pointid from `i b`
    - `i b` information will be like a order list. If delete a pointid in the previous, the next point will just add in the end of the idlist.
- run debug
    - `r`
    - use `p variable` will show the variable information
- gdb controler
    - `n` for next line, one step debugging
    - `s` for step into(one step)
    - `set args [parameter]` for command line args
- finish function and continue
    - `finish` for finish function
    - `c` for continue until program exit
- quit gdb
    - `q`
