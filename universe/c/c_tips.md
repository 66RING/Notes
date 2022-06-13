---
title: C中一些容易疏忽的问题
author: 66RING
date: 2021-06-06
tags: 
- c
mathjax: true
---

## C中一些容易疏忽的问题

- 数组作为形参时退化为指针，即`sizeof(arg) = 8`，不再是原先数组的大小
	* 可以使用引用(CPP中)，`f(int (&arg)[])`


## 字符串定义方式的问题

`char* a`和`char a[]`类型不同，导致的结果不同：

- `extern char *a`认为存储单元是地址的值
- `extern char a[]`认为存在单元是数组元素，但是用的时候a是地址

```c
/*a.c*/
#include <stdio.h>
extern char a[];
extern char *b;
extern unsigned int a_address();
extern unsigned int b_int();
int main()  {  
    if(*(unsigned int*)a==a_address())  
		// a[]数组的值，保存了a指针地址
		printf("指针a被错误的认为是数组\n");  
    if((unsigned int)b==b_int())  
		// *b保存的就是指针的值
        printf("数组b被错误的认为是指针\n");  
    return 0;  
}  

/*b.c*/
char *a="abcdefgh";  
char b[]="ABCDEFGH";  
				      
//获取指针a本身存储的内容，并转化为无符号整数  
unsigned int a_address() {
	return (unsigned int)a;  
}  
//32为系统，所以获取数组前四个字节转化为无符号整数  
unsigned int b_int() { 
	return *(unsigned int *)b;  
}
```


## 64位和32位的问题

系统的内存布局，变量传递方式等都可能不同，导致64位系统和32位系统表现的结果不同。

```c
void f()
{
	// 32 位系统中 a在低地址，PI在高地址，从而a[16]会修改到PI的值
	const float PI = 3.14159;	
	int a[16];	
	a[16] = 3;
	printf("PI = %f\n",PI);
}
```

