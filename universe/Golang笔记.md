---
title: Golang
date: 2019-08-05
---

非零基础笔记，掌握c的基础上记录

### GO mod的使用
``` go
go mod init
go mod tidy
"当然要设置好代言"
"https://goproxy.io"
```

### 基本

``` 
package main
import "fmt" //import了就必须使用
// 不用写；
func main(){ //这个括号不能换行
		a := 10 //:=是自动识别类型的
		b,c := 20,30
		var b int //var 变量名 类型
		fmt.Println() //自动换行
		fmt.Printf() //格式化输出，同c
		 
}

```

### 匿名变量

``` 
i,_ := 10,20 // _表示匿名变量 配合函数返回值才有优势  多返回值+占位
```

### 常量

``` 
const a int = 10
aonst b = 20 // 没有:= ,也自动推到类型
```

### 多个变量或常量到定义

``` 
var(
		a int
		b float64
)

const(
		a int = 1
		b float64 =1.2 //或者直接=让它自动推到类型
)
```

### iota枚举

``` 
const(
		a = iota
		b = iota
		c = iota
)
		1.iota常量自动生成器，每一行自动加一
		2.iota给常量赋值
		3iota遇到新的const时重置为0
		4.可只写一个iota
		const	(
		  a = iota
		  b
		  c
		)
		5.如果同一行，值都一样
		const(
				i = iota
				j1,j2,j3 = iota,iota,iota  //一样
				k = iota
		)
```

### 类型别名

``` 
type i int //给int取个别名i
  ^
```

### if特性

``` 
//支持初始化
if a:=0;a<10{
		^			^	分号隔开
}
```

### switch-case-fallthrough

​								^跳出，相当于break
​		支持一个初始化语句,

### range迭代

``` 
str:="abd"
for i,data := range str{ //第一期是位置str[]，第二个是值

}
```

### 函数

``` 
func 函数名(a int,b string参数列表)(a1 type1,a2 type2返回类型){
	函数体
	return v1,v2
}

//不定参数列表
func funcName(a ...int){
							...int^不定参数类型，必须放在最后
							像一个列表
							可以for i,data :=range a{
							
							}
}
```

### 返回值

``` 
//常用写法
func name()(res int){
		res = 666
		return  //有返回值必须返回嘛，所在go中可以这样写
}
```

### 函数也是一种数据类型

``` 
//故可以用type重命名
type newname func(a,b int) int{
}
var a newname
a = (1,2)

//回调函数，例
func cala(a,b int, fuc newname) int {
	fuc(a, b)
} //                 ^多态了
func main(){
	cala(1,2,add)
}

```
###匿名函数
```
f1 := func(){ //没有参数，闭包可捕获到外层到变量，影响到外面
			//不管变量到作用域，只要闭包还在使用他，变量就还在
}
//匿名函数定义了就要有东西来接
//不然这么写
func(){
}()
  ^这个括号表示传进去到参数，然后直接调用

```
###defer
```
1.延迟调用，函数结束前(那一刻)调用
2.先写到后调用
3.多个defer调用时，哪怕其中有个出了错，其他仍然会执行
4.相当于先按顺序读，然后默默记住，再倒序输出
```
###获取命令行参数
```
1.导入包
import "os"

2.接收传入的参数 //全都是以字符串形式
str := os.Args  <这个表示
tips:可用不定参数接，可用range迭代接
```
###导入包
```
同一个目录包名必须一样
同一个目录调用别的文件的函数无需包名
不同目录包名不一样
调用的别的包的函数首字母必须大写

1.除传统写法还可
import(
	""
	""
) 

2.起别名
import 别名 "fmt"

3.  .操作
import . "fmt" //调用函数无需通过包名

4.忽略次包名
import _ "fmt"
这个操作时为了调用包中的init函数
```
###init函数
```
导入一个包就先执行一个包的init函数
func init(){
}

所以 import _ "" 操作的目的时调用init函数而不用其他函数
```

### 数组

``` 
//指定下标初始化
d := [5]int{2: 10,4: 2}
						^指定的下标
						
//数组比较
比较类型和每个元素
//同类型的数组可以赋值

//数组做函数参数->拷贝，值传递

//数组指针防止值传递^
p *[7]int
(*p)[0]  <-注意写法

```

### 随机数的使用

``` 
1. import
2. 设置种子 //种子参数一样产生的随机数一样
```

### 切片

```
array := [...]int{10, 20, 30, 0, 0}
slice := array[low:high:max]  //切片
low->起点
high->终点 //不包括
长度=high - low
容量=max - low  //意义不明

创建方式：
1.传统方式
	s1 := []int{1,2,3} //自动推到类型同时初始化
2.make函数
	s2 := make([]int, len, cap)
						^切片类型      ^默认不写和len一样

切片操作
类似python
操作某个元素和数组操作一样
append函数，向后追加，自动以两倍容量扩容
copy函数，copy(目标切片， 原切切片)

！！！切片做函数参数！！！
引用传递，会影响到外面
```

### 切片和数组的区别&&关系

```
数组长度固定，切片[]里面为空或...长度可以不固定
a := [7]int{}
s := []int{} <-	切片

对切片操作会影响到底层数组
```

### map

```
格式： map[keyType]valueType
创建方式
1. var m1 map[int]string

2. m2 := make(map[int]string) 

3. m2 := make(map[int]string, len) //指定容量，自动扩容，提前分配提高效率 
```

### int和string的转换

```
#string到int  
int,err:=strconv.Atoi(string)  
#string到int64  
int64, err := strconv.ParseInt(string, 10, 64)  
#int到string  
string:=strconv.Itoa(int)  
#int64到string  
string:=strconv.FormatInt(int64,10) 

同类型之间转换，比如int64到int，直接int(int64)即可；
```

### 接收复杂的类型一般自定义结构体

```go
因为http只能传字符串，所以前端要传json的话要先把json转换成json字符串
……
```

