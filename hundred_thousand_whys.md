---
title: Hundred Thousand Whys
date: 2020-9-17
tags: tips, tools
---

## Why rust say it want to replace c++

Rust has a specfial  *Memory Management* , which is an **COMPILE-TIME memnory management base on Ownership, Borrowing and Lifetime** .

- C family: manually manage pointers and  **trust programmer access them and free correctly**. Otherwise wil cause run-time errors.
- Java: Run-time GC
- Rust: Ideally not to manage pointers manually, it has a compile-time borrow check.


## Diff in DOS, Mac and Unix file

The old Teletype machine used two character to satrt a new line. One to move the carriage(return \<CR\>) back to first position.

Back in the day, storage was expensive. Some people decided they did not need two character for end-of-line.

- The Unix people decided they use `Line Feed` only for end-of-line
- The Apple people decided they use `<CR>`
- The Window forks decided to keep the old `<CR><LF>`


