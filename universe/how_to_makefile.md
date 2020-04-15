---
title: How to makefile
date: 2020-2-26
tags: makefile, tools
---

# How to makefile
## Compile process
How dose a project turn into an excutable file? There are four step:
- Preprocessing
    - Unfold all include libs, macro, head. And produce `.i` file.
    - (gcc/g++ commands)`-E`
- Compilation
    - The `.i` file produce an `.s` file(assembly code).
    - `-S`
- Assemble
    - The `.s` file produce an `.o` file(object file).It is a binary file.
    - `-c`
- Linking
    - Linking the `.o` file produce an excutable file.

Whole example
``` 
gcc -E halo.c -o halo.i  
gcc -S halo.i -o halo.s  
gcc -c halo.s -o halo.o  
gcc -o halo.o -o halo
```

## Why makefile
As we see, compile a file include so much process.Though `gcc halo.c -o halo` can do the same thing as above, you need to make sure all the `.c` file are in the directory. 
And what if a thousand file to compile?

## Lets start
### Format
``` makefile
target: dependencies
    command

^ before command IS A TAB, and make sure a new line at the end
```
We should write "from rear to front". Which means write final target command first and then write dependencies commandS

### Usage
``` sh
make target...
```

If didn't specify target, run the first target as default. That is why should we wirte main object at first.
