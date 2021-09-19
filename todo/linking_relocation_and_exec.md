---
title: 链接、重定位与执行TODO
date: 2021-04-03
tags: 
- compile
- OS
mathjax: true
---

# Preface


# Overview

# linking

- linker
- object file`.o`
	* module
	* section
	* symbol table
		+ symbol entry
- static library
	* `.a` archive
	* why archive
		+ wast space
		+ management
	* different from module
- link with static libraries
	* left to right
	* order related
		+ algo: E(executable), U(undefine), D(defined)

# relocation

assign run-time address

assembler:

- relocation entry
	* `rel.text`
	* `rel.data`
	* when encounter "unknown"
	* tell linker

```
typedef struct {
	long offset; 			// section offset of the reference
	long type: 32,  		// Relocation type
			symbol: 32; 	// Symbol table index
	long addend; 			// Constant part of relocation
} Elf64_Rela;
```

- refaddr = ADDR(s) + r.offset: 当前地址
- ADDR(r.symbol): 对应的符号表中的符号所在地址

- type
	* rel `R_x64_64_PC32`
	* abs `R_x64_64_32`

- relative: 
	* ADDR(r.symbol) - refaddr
- abs


# exec

- program header table: `prog.`


# 链接











