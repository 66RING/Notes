---
title: GDB usage
date: 2020-2-25
tags: 
- tools
- gdb
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
- breakpoint
    - use `b <LINENUM>` to add breakpoint, such as `b 1` add breakpoint at line 1
    - or use `b <FUNCTIONNAME>` to add breakpoint, such as `b main` add breakpoint at main function
    - `b <[file:]linenum|[file:]funciont>`
    - `tb[reak]` for a temporary breakpoint
- Show breakpoint information
    - `i b` i for information, b for breakpoint
    - `clear [[file:]function|[file:]lennum]` to clean all breakpoint
    - `rb[reak] REGEXP` set breakpoint for all function matching REGEXP
    - `ignore N COUNT` ignore bp 1 for COUNT times 
- **Inspect**
    * use `print, inspect or p [OPTION...] [EXP]` to print value of expression EXP
        + use `p a` will show information of variable `a` 
        + `p [/FMT]` print with specified format
        + `p ARRAY[@len]` print arrary with len element
        + `p ['file'::]var` print var in specified file
- Delete breakpoint
    - `d <BREAKPOINTNUM>` break point num from `i b`
    - `i b` info of breakpoint, information will be like a order list. If delete a pointid in the previous, the next point will just add in the end of the idlist.
- Run debug
    - `r [args...]`
- Gdb controler
    - `n [N]` **for next line**, one step debugging
    - `s` **for step into**(one step)
    - `u|until [LINENUM|FUNC|ADDR|if condition]`
    - `set args [parameter]` for command line args
    - `return <value>` finish function and return `<value>`
    - `call <function>` call specified function
- Finish function and continue
    - `finish` for finish function
    - `c` for continue until program exit or breakpoint
- Quit gdb
    - `q`


## Tips

- You can write shell script in gdb
    * `(gdb) shell <cmd>`
- Set log
    * `set logging on`, which will copy output to file `gdb.txt`
- **Watchpoints**, Catchpoint... and so on
    * You can use a watchpoint to stop execution whenever the value of an expression **changes**.
    * `watch *<addr>` or `watch <value>`
    * `info watchpoints` to print info of watchpoints
- Debug a running program
    * `gdb -p <pid>`
- Debug a core file
    * `gdb <binary file> <core file>`
    * System will no generate a core file by default. Because typically core files are very big. You can use `ulimit` command to set unset the limit.
- 自定义脚本

```cpp
define print_list
set $list=$arg0
while($list)
printf "%d\n",$list->val
set $list=$list->next
end
printf "\n"
end
```


