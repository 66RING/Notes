---
title: Golang提高
date: 2020-7-23
tags: go
---

## GoTest

使用go test，文件名格式有要求`XXX_test.go`


## 杂项

###  字符串变量结构

- 不同于C语言字符串以`\0`结尾，golang字符串中的字符可以是任何字符，因为它的结构为: `| data | len |`
- 因此你可以像这样`str[2]`读取字符串内容，但不能修改它


### 切片

slice有三个部分

- data(元素存哪里)
- len(存了多少)
- cap(可以存多少)
- 因此slice结构为 `| data | len | cap |`

``` go
var ints[]int = make([]int, 2, 5)
ints = append(ints, 1)

ints: | data | 3 | 5 |

| 0 | 0 | 1 | 0 | 0 |
```

当使用new创建字符串切片如`ps := new([]string)`，会分配一个slice的三部分结构`| data=nil | 0 | 0 |`，返回值就是slice的起始地址。但它不负责底层数组的分配，所以ps指向nil

通过`append(*ps, "hello")`添加元素，它就会slice开辟底层数组(上节说的string结构)

但是slice不是必须指向数组的开头，因为我们可以把不同是slice关联到同一个数组`arr[a:b]`，他们会共用底层数组

``` go
arr := [7]int{0, 1, 2, 3, 4, 5, 6}
// arr 是一个长度为7的int数组
var s1 []int = arr[1:4]
var s2 []int = arr[3:]

//左闭右开，则s1结构为
| data | 3 | 6 |  // 可以继续添加元素
// s2为
| data | 4 | 4 |  // 如果继续添加元素，则开辟新数组并拷贝原数据
```

slice扩容的步骤：

- 根据slice的扩容规则，预估
    * 如果扩容前(oldLen)翻倍还是小于所需最小容量(cap)，则新容量(newCap)等于最小容量
    * 否则
        + 如果oldLen < 1024 , 直接翻倍newCap = oldCap x 2
        + 否则扩1/4，newCap = oldCap x 1.25
- 分配内存
    * golang的内存管理模块会提前申请好一部分常用规格的内存，如8、 16 ...字节
    * 然后分配最接近需求的内存


### 内存对齐

[building](https://www.bilibili.com/video/BV1Ja4y1i7AF)

因此好的go程序结构体字段的顺序也是有讲究的


### 闭包

go将作为参数、函数返回值、绑定到变量的函数称为 **Function Value** ，Function Value本质上是个指针，但不直接指向函数入口，而是指向一个`runtime.funcval`结构体，这个结构体里只有一个地址，就是这个函数的入口地址。

``` go
type funcval struct{
    fn uintptr
}
```

编译器会为同一个函数的Function Value指定相同的funcval。

Golang使用funcval接口体包装函数地址的原因：为了处理闭包的情况。举个闭包的例子

``` go
func create() func()int{
    c:=2
    return func() int {
        return c
    }
}
```

c这样的变量称为 **捕获变量** 。闭包对象在运行时是才创建。当一个变量接受调用时，如`f1 := create()`会在堆中创建一个funcval结构体和捕获变量列表。当另一个变量调用时又生成另一个funcval和捕获变量列表。这样每个闭包的状态(捕获变量)可能有所不同，这就是为什么称闭包为有状态的函数。

有了funcval的结构就可以通过funcval的指针找到函数入口，通过与funcval的偏移量找到捕获变量。

#### 变量逃逸

考虑捕获的变量处理初始化外还被修改的情况

``` go
func create() (fs [2]func()){
    for i:=0; i<2; i++{
        fs[i] = func(){
            fmt.Println(i)
        }
    }
    return
}

func main(){
    fs := create()
    for i:=0; i<len(fs); i++{
        fs[i]()
    }
}
```

结果输出都是2，原因如下。

在`create()`函数中，因为i被闭包捕获，局部变量i改为堆分配，在create函数的栈中值保存i的地址(&i)。第一次for创建funcval和捕获列表(i的地址)，这样闭包函数就和外层函数操作同一个变量，第二次for循环仍是funcval和i的地址。因为操作的是同一个变量，而闭包函数捕获的是i的地址，所以两次输出的结果其实都是create堆分配中的i(2)。

闭包导致的局部变量堆分配就是变量逃逸的一种场景。闭包这么做是为了保持捕获变量在外层函数与闭包函数中的一致性。


### defer

defer会在函数结束前倒序执行。
here <-


## Context

### 常见的控制并发的两种方式

- 1. WaitGroup
    * 使用场景：goroutine同时做一件事，都做这件事的一部分，只有全部goroutine做完这件事才完成
        ``` go
        var wg sync.WaitGroup
        wg.Add(2)
        go func(){
            // ...1...
            wg.Done()
        }
        go func(){
            // ...2...
            wg.Done()
        }
        wg.wait()  // 会等待两个go里的wg都Done，即go执行完毕
        ```
- 2. Context
    * 需要主动停止goroutine
        + 1. channel + select
            ``` go
            stop := make(chan bool)
            go func(){
                for{
                    // 无限循环的执行某些任务...
                    select {
                    case <- stop:
                        // 要停止了
                        return
                    default:
                        // 继续执行、sleep一下等
                    }
                }
            }
            // 执行了很多业务，然后想要停止go了
            stop <- true
            ```
        + 但是如果存在多个goroutine或goroutine内又有goroutine时channel+select的方法不再适用。因为业务可能很复杂
        + 2. context(上下文)：使用context跟踪goroutine以便控制，所有基于这个context或衍生的子context都会收到控制通知
            ``` go
            func worker(ctx context.Context, args){
                go func(){
                    for{
                        select{
                        case <- ctx.Done():
                            // ...
                            return
                        default:
                            // ...
                        }
                    }
                }
            }
            ctx, cancel := context.WithCancel(context.Backgroud())  // 返回的cancel做控制
            go worker(ctx, node1)
            go worker(ctx, node2)
            go worker(ctx, node3)
            go worker(ctx, node4)
            cancel() // 就可这样方便的控制
            ```


### Context接口

- 1. `Deadline()(deadline time.Time, ok bool)`，获取截止时间
    * 第一个返回值是截止时间，到达这个时间后Context会自动发送取消请求
    * 第二个返回值表示有没有设置截止时间
    * 如果需要取消，要调用取消函数
- 2. `Done() <-chan struct{}`
    * 返回一个只读的chan，类型为struct{}，如果返回的chan可以读取，则意味着父Context已经发出取消请求
- 3. `Err() error`
    * 返回取消的错误原因
- 4. `Value(key interface{}) interface{}`
    * 获取Context上绑定的值，一个键值对，这个键值对一般是线程安全的(保证多个go访问是安全的)


### Context的继承衍生

使用context包提供的With系列函数通过父Context可以衍生出很多子Context

- `func WithCancel(parent Context) (ctx Context, cancel CancelFunc)`
    * 返回子Context以及一个取消函数来取消Context
- `func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)`
    * 需要截止时间作为参数
- `func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)`
    * 不同于Deadline，Timeout表示多少时间后取消
- `func WithValue(parent Context, key, val interface{}) Context`
    * 生成一个绑定了一个键值对数据的Context


### WithValue传递元数据

通过Context我们也可以传递一些必须的元数据，这些数据会附加在Context上以供使用

``` go
ctx, cancel := context.WithCancel(context.Background())
valueCtx := context.WithValue(ctx, key, "value1")
go worker(valueCtx)
```

这样就可以通过valueCtx.Value(key)来获取值了。使用WithValue传值，一般是必要的值


### Context使用原则
 
- 不要把Context放在结构体中，要以参数的方式传递
- 以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。
- 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO
- Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
- Context是线程安全的，可以放心的在多个goroutine中传递



