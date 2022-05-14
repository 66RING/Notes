---
title: lex和yacc使用
author: 66RING
date: 2022-05-03
tags: 
- compiler
mathjax: true
---

# Abstract

- 使用lex工具做词法分析
- 使用yacc做语法分析


# Lex Usage

##  编译运行

- `flex <file.l>`默认生成lex.yy.c文件
- `gcc -lfl lex.yy.c`


## lex源程序

三段式:

```
声明部分
%%
规则部分
%%
用户自定义函数
```

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


# Yacc Usage

## 编译运行

- `yacc -d -b <PREFIX> <file.y>`
	* `-d`可以产生一个posix标准的头文件
	* `-b`可以执行产生的文件的前缀
	* 详见`yacc -h`
- `gcc lex.c yacc.c -o outfile`


## yacc源程序

同样三段式(`[]`表巴克斯范式的可选)

```
[
%{
C声明
%}
符号定义
]
%%
规则
	产生式

[%%
用户自定义函数
]
```

词法分析器识别**记号流交给语法分析器**

```yacc
%{
	#include<ctype.h>
%}

%token NUMBER	// 词法分析器返回的
%left '+','-'	// 特殊的终结符：左结合，同理%right右结合
%left '*','/' 	// 放在下面的规则优先级高，下高上低左高右低

%%

expr :expr'+'expr {printf("识别加法");}
	 |expr'-'expr {printf("识别减法");}
	 |expr'*'expr {printf("识别乘法");}
	 |expr'/'expr {printf("识别除法");}
	 |'('expr')' {printf("识别括号");}
	 |NUMBER {printf("识别数字");}
	 ;

%%

void yyerror(const char *str){
    fprintf(stderr,"error:%s\n",str);
}

int yywrap(){
    return 1;
}

int main() { return yyparse();} 	// yyparse调用语法分析器
									// 内部自动调用词法分析器yylex
```

需要`yyerror()`和`yywrap()`将yacc补充完整

- `yyerror()`用于处理产的错误
- `yywrap()`用于处理是否继续打开其他文件，返回1表示结束，返回0表示继续打开其他文件

需要注意的是，如果要处理多条命令，需要在规则中显式指出，即**添加左递归**，否则将会`syntax error`，如

```
commands: // 空
		| commands command

command:STRING { printf("识别string"); }
	 |NUMBER { printf("识别数字"); }
	 ;
```

## tips

**语法分析教会了我什么**: 遇到递归可以从终结符开始分析。

理解和妙用左递归

- `%left`, `%right`左结合token，右结合token
- 优先级左高右低，上高下低
- yacc是自底向上语法分析，可以处理左递归的情况
	* 左递归可以用于处理可变"参数"
- 为了结构清晰可以使用别名，即新增一条产生式

下面的例子中用到了左递归接收可变的"参数"用逗号隔开。使用"别名"，让结构更清晰

```yacc
table_field: ID;
table_fields: table_fields','table_field
			| table_field
			;
```


# lex与yacc结合

yacc可以产生一个头文件，如`y.tab.h`，里面会包含语法分析定义的token。在lex中引入该头文件`#include "y.tab.h"`然后就可以在识别到记号后返回相应的token来告知语法分析器。

形如：

lex文件：

```lex
%{
 #include"y.tab.h"
%}

char [a-zA-Z]
chars {char}+
digit [0-9]
digits {digit}+

%%
{digits} { return NUMBER; }
{chars} { return STRING; }
%%
```

yacc文件：

```yacc
%{
	#include<ctype.h>
	#include<stdio.h>
%}

%token NUMBER STRING

%%
commands: // 空
		| commands command
command:STRING { printf("识别string"); }
	 |NUMBER { printf("识别数字"); }
	 ;
%%

void yyerror(const char *str) { fprintf(stderr,"error: %s\n",str); }

int yywrap() { return 1; }

int main() { 
 	// yyparse调用语法分析器
 	// 内部自动调用词法分析器yylex
	return yyparse();
}
```



