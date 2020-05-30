---
title: emacs快速入门
date: 2020-5-29
tags: emacs, linux, editor
---

## Emacs细节

- major mode和minor mode
    - 打开一个文件时会有默认的mode激活，这个默认的mode就是major mode
    - minor mode在配置文件中，状态栏不会显示，`c-h m`显示打开的minor mode
- org mode标签TODO/DONE
    - `c-t/c-s`

## Emacs基本操作

- M for meta, Alt or Command(MAC)
- S for shift
- C for control
- s for super

- Move
| Key  | action   |
|------|----------|
| c-f  | forward  |
| c-b  | backward |
| c-p  | previous |
| c-n  | next     |
| c-a  | ahead    |
| c-e  | end      |
| <++> | <++>     |

- Action
| Key  | action                   |
|------|--------------------------|
| c-w  | cut                      |
| m-w  | copy                     |
| c-y  | yank(paste)              |
| m-<  | begin of file            |
| m->  | end of file              |
| c-k  | del to end of line(kill) |
| <++> | <++>                     |
| <++> | <++>                     |
| <++> | <++>                     |

- Edit
| Key              | action                                 |
|------------------|----------------------------------------|
| C-g              | 中断所有操作                           |
| C-x C-f          | 打开文件                               |
| C-x C-s          | save                                   |
| C-h key/var/func | help for key/var/func                  |
| C-x C-e          | 执行表达式，根据括号范围决定运行的指令 |
| <++>             | <++>                                   |
| <++>             | <++>                                   |
| <++>             | <++>                                   |
| <++>             | <++>                                   |
| <++>             | <++>                                   |
| <++>             | <++>                                   |

- Search
| Key  | action       |
|------|--------------|
| c-s  | search below |
| c-r  | search above |
| <++> | <++>         |
| <++> | <++>         |



## Elisp

括号括起来是表达式，表达式第一个参数是函数，后面是参数。如
- `(+ 2 2)`，+的函数，结果是2+2

括号也起到了代码块的作用

| 指令                                 | 功能                                          |
|--------------------------------------|-----------------------------------------------|
| (setq var [value])                   | 定义变量（赋值）                              |
| (message var)                        | 格式化输出，同C                               |
| (defun func)                         | 定义函数                                      |
| (interactive)                        | 声明函数是交互式的函数，<M-x>可以找到这个函数 |
| (global-set-ket (kbd "<key>") 'func) | 定义快捷键                                    |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |
| <++>                                 | <++>                                          |



