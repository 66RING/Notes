---
title: How to makefile
date: 2020-2-26
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

