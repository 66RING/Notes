---
title: lex词法分析工具使用
author: 66RING
date: 2022-05-03
tags: 
- compiler
mathjax: true
---

# Abstract

使用lex工具做词法分析，不需要手撸状态机。


# Usage

##  编译运行

- `flex <file.l>`默认生成lex.yy.c文件
- `gcc -lfl lex.yy.c`


## lex源程序

三段式:

```
%{
	辅助性C声明
%}

辅助定义正规式
<名> <正规式>

%%
<规则> <事件>		# 一行一个，可以{后换行
<规则> <事件>
<规则> <事件>
%%

C代码
```

1. "辅助定义和规则事件段的内容就会自动转换成响应的C代码插在中间"
2. 规则中可以使用上面的辅助定义，事件可以使用第一段的C声明

e.g. 可以看到用到辅助定义的地方都"{}"括起来了

```lex
%{
#define ID 0
#define NUMBER 1
%}

char [a-zA-Z]
digit [0-9]
digits {digit}+
optional_fraction ("."{digits})?
optional_exponent (E[+-]?{digits})?

%%
{char}({char}|{digit})* {
		printf("recognize token %s, len %d\n", yytext, yyleng);
		return ID;
	}
{digits}{optional_fraction}{optional_exponent} {
		printf("recognize number %s, len %d\n", yytext, yyleng);
		return NUMBER;
	}
%%

int main() {
	printf("type: %d", yylex());
	return 0;
}
```

## tips

1. lex大小写不敏感
2. 先定义的规则先匹配
3. 内置的变量
	- `yytext`表示当前匹配到的内容
	- `yyleng`表示当前匹配到的内容的长度
	- `yylex()`，即主函数，扫描输入，匹配规则，如果规则中有return则返回
4. **使用辅助定义正规式时要用{}**
5. **""**中的内容匹配字面量，类似转意
	- 如要匹配`.`的情况
6. 可以在规则最后添加一个`.*`规则作为未定义行为，因为lex最长最先匹配的规则可能会导致字串被匹配到，如`create"abc`会把create匹配，但其实这应该是undifine的



