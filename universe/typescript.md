---
title: TypeScript
date: 2020-3-2
tags: typescript, javascript
---

# TypeScript

## 使用
typescript是编译型,编译成javascript(解释型).

- 安装
    - `npm install -g typescript`
- 编译
    - `tsc hello.ts`
- 编写react时以.tsx为后缀

## 强类型
### 指定类型
``` typescript
var str:string = "1";
var num:number = 1;
var bool:boolean = true;
var un:undefined = undefined;
var nil:null = null;
var a:any;
a = 1;
a = "1";

var func1 = function():void{
     
}

var func2 = function():number{
   return 0 
}
```

没有赋值,表示是any值,以后不管怎么赋值都是any值的类型
``` typescript
var a2;
a2 = 1;
a2 = "1";
```

联合类型,方法只能调用类型公共的方法
``` typescript
var strnum:string|number = "1";
// strnum.length() 就不行
strnum.toString()
```

### 接口
类似java的接口,'子类必须实现接口'
``` typescript
interface Inf{
    name:string
}

var obj:Inf;
obj = {name:"ring"}  // 没有赋值会报错
```

可选属性
``` typescript
interface Inf{
    name:string,
    age?:number  // 可选类型
}

var obj:Inf;
obj = {name:"ring"}  
```

不确定数量属性
``` typescript
interface Inf{
    name:string,
    age?:number, // 可选类型
    [propName:string]:any  // 不确定数量的propName(strng类型),他的值是any
}

var obj:Inf;
obj = {name:"ring", age:10, sex:man}  
```

只读属性
``` typescript
interface Inf{
    name:string,
    readonly age:number  // 只读属性
}

var obj:Inf;
obj = {name:"ring", age:10, sex:man}  
```

### 数组
类型方括号
``` typescript
var arr:number[] = [1, 2, 3]
var arr2:any[] = [1, 2, 3]
```

泛型定义法
``` typescript
var arr:Array<number> = [1, 2, 3]
var arr2:Array<any> = [1, 2, 3]
```

接口定义法
``` typescript
interface IArr{
    [index:number]:number
}
var arr:IArr = [1, 2, 3]

// 类型除了是内置的类型, 还可以是接口的类型
interface Istate{
    name:string,
    age:number
}

interface IArr{
    [index:number]:Istate
}
var arr:IArr = [{name:"ring", age:20}]
var arr:Array<IArr> = [{name:"ring", age:20}]
var arr:Istate[] = [{name:"ring", age:20}]
```

### 函数
声明式函数
``` typescript
// 函数名 参数名:类型  :返回类型
function func(name:string, age:number):number{
    return age
}

var age:number = func("ring", 20)
```

函数参数不确定
``` typescript
function func(name:string, age:number, sex?:string):number{
    return age
}

var age:number = func("ring", 20, "man")
```

参数默认值
``` typescript
function func(name:string, age:number, sex?:string="man"):number{
    return age
}

var age:number = func("ring", 20, "man")
```

表达式类型函数
``` typescript
var func = function(name:string, age:number):number{
    return age
}

var age:number = func("ring", 20)
```

表达式类型函数变量的约束规范
``` typescript
// 变量  约束  类型
var func:(name:string, age:number)=>number = function(name:string, age:number):number{
    return age
}

var age:number = func("ring", 20)

// 或者使用接口
interface Istate{
    (name:string, age:number):number  // (参数):返回类型
}
var func:Istate = function(name:string, age:number):number{
    return age
}

var age:number = func("ring", 20)
```

采用重载的方式支持联合类型的函数关系
``` typescript
function func(value:string):string;
function func(value:number):number;
function func(value:string|number){  // 这是上面的实现
    return value
}
let a:number = func(1)
let b:string = func("1")
```

### 类型断言
在联合类型的情况下, 调用的函数要是共有的函数, 不然会报错, 所以需要进行断言

<类型>值 或 值as类型
``` typescript
function func(name:string|number){
    return (<string>name).length()
    // 或者return (name as string).length()
    // 只能转换成联合类型中存在的类型
}
// !! jsx/tsx中必须采用后一种 !!
```

### 类型别名
``` typescript
type strtype = string|number;
var str:strtype = "10"
str = 10
```

对于接口
``` typescript
interface muchType1{
    name:string
}
interface muchType2{
    age:number
}

type muchType = muchType1|muchType2
var obj1:muchType = {name:"ring"}
var obj2:muchType = {age:1}
var obj3:muchType = {name:"ring", age:1}  // 一种或多种或全部
```

限制字符串的选择
``` typescript
type sex = "man"|"woman"
function func(s:sex):string{
    return s
}
func("man")
// func("an") 就不行
```

### 枚举
使用枚举可以定义一些有名字的数字常量
``` typescript
enum Days{
    Sun,  // 默认第一个取值为0
    Mon,  // 后面的一次累加
    Tue,
    Wed,
    Thu,
    Fri,
    Sat,
}
console.log(Days.Sun)
```

typescript枚举出来会自动赋值外, 同时会对枚举值到枚举名的反向映射, 就是变成了双向映射
``` javascript
// ts编译成js后的结果
// 即Days[0] = "Sun"
var Days;
(function (Days) {
    Days[Days["Sun"] = 0] = "Sun";
    Days[Days["Mon"] = 1] = "Mon";
    Days[Days["Tue"] = 2] = "Tue";
    Days[Days["Wed"] = 3] = "Wed";
    Days[Days["Thu"] = 4] = "Thu";
    Days[Days["Fri"] = 5] = "Fri";
    Days[Days["Sat"] = 6] = "Sat";
})(Days || (Days = {}));
console.log(Days.Sun);
```

### 类的修饰符
public private protected
``` typescript
class Person{
    name = "ring",
    private age = 20,
    protected say(){
        console.log("halo")
    }  
}

// 没有修饰的话, 类中的属性后方法默认是public
// private 属性只能在类中被访问
// protected 只能在类及其子类中访问
```

类的继承
``` typescript
class Child extends Person{
    callParent(){
        super.say()  // super能拿到父类的对象, 所以能访问父类公开的属性和方法
        // 类中也能访问父类protected的方法, 类外就不行
    }
}
```

静态方法, 可以通过类名调用
``` typescript
class Child extends Person{
    callParent(){
        super.say()  
    }
    static test(){
        console.log("halo")
    }
}

console.log(Child.test())  // 但在静态方法中不可以使用this
```

### 泛型
泛型是指在定义函数, 接口或类的时候, 不预先指定具体类型, 而在使用的时候再指定类型的一种特性.
也可以帮助我们限定约束规范
``` typescript
// 用T来统一类型, T可以是任何名字
function func<T>(len:number, value:<T>):Array<T>{
    let arr = []
    for(var i=0;i<len;1++){
        arr[i] = value
    }
    return arr;
}

var strArry: string[] = func<string>(3, '1')  // 制定类型为string
var numArry: number[] = func(3, 1)  // 右边也可以不指定类型, 因为左边要求就收的是number, 所以它会反推
```

接口采用泛型
``` typescript
interface Istate{
    <T>(name:string, value:T):Array<T>
}

let func:Istate;
// 接口约束函数返回值
func = function<T>(name:string, value:T):Array<T>{
    return []
}

// 调用时指定类型
var strArry:string [] = func("ring", "2020")  // 2020就指定了类型是string
var numArry:number [] = func("ring", 2020)  // 同理
```









