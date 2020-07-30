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

``` go
func A(){
    defer B()
    // do something
}
```


#### Go 1.12

在go1.12中编译后伪代码结构为：

``` go
func A(10){
    r = deferproc(8, B)
    if r>0 {  // if panic
        goto ret
    }
    // do something
    runtime.deferreturn()
    return
ret
    runtime.deferreturn
}
```

- deferproc把要执行的函数信息保存起来，称之为 **defer注册** 
    * `func deferproc(size int32, fn *funcval)`, size指出defer函数的参数加返回值共占多少空间
- runtime.deferreturn执行注册的defer函数
    * 这样的先注册后调用来实现延迟执行
    * defer信息会注册到一个链表，而当前执行的goroutine会在堆中创建一个`_defer`结构体，其中一个属性持有链表的头指针，新插入链表头，因此是后进先出，defer倒序执行

``` go
type _defer struct{
    siz     int32    // 参数和返回值共占多少字节
    started bool     // 是否已执行
    sp      uintptr  // 调用者栈指针
    pc      uintptr  // 返回地址
    fn      *funcval // 注册的函数
    _panic  *_panic 
    link    *_defer  // next_defer
}
```

Go语言会预先分配一块大小的\_defer池，需要defer时选择合适的取出，否则再进行堆分配。这样避免了频繁的堆分配//回收。

这个go1.12版本的defer存在的问题就是慢：
- 1. \_defer在堆分配，使用时要在堆和栈间来回拷贝
- 2. 链表本身的操作就比较慢


#### go1.13的优化策略

在go1.13中编译后的伪代码结构为：

``` go
func A(){
    var d struct{
        runtime._defer
        i int
    }
    d.siz = 0
    d.fn = B
    d.i = 10
    r := runtime.deferprocStack(&d._defer)
    if r>0 {
     goto ret
    }
    //do something
    runtime.deferreturn()
    return
ret:
    runtime.deferreturn()
}
```

1.13中通过增加d结构体这样的局部变量把defer信息保存在当前函数栈帧的局部变量区域，再通过deferproc把d这个结构体注册到defer链表中

减少的defer信息的堆分配，\_defer也新增了`heap bool`的字段来表示是否为堆分配


#### go1.14中defer的优化策略

通过在编译阶段插入代码，把defer的执行逻辑展开在所属函数中，从而不需要创建\_defer结构体和defer链表

``` go
func A(i int){
    defer A1(i, i*2)
    if(i > 1){
        defer A2()
    }
    return
}
```

编译时defer函数如A1，所需的参数以局部变量的形式保存在所属函数中，然后在函数末尾插入`A1(保存的局部变量)`

但像A2这样执行阶段才知道是否需要执行的defer函数怎么能直接插入呢？

使用一个标识位`var df byte`每一位表示一个defer函数是否会执行，使用或运算来修改df标记位的信息。因此上述函数对应的编译后伪代码为：

``` go
func A(i int){
    var df byte
    //defer是否需要执行的标记位
    var a, b int = i, i*2 
    var m, n string = "hello", "ring"
    //局部变量保存defer函数的参数
    df |= 1 //defer函数A1必定会执行，所以直接
    if i>1 {
        df |= 2  //defer函数A2可能会执行，runtime才知道
    }
    if df&2 > 0 {
        df = df&-2 //-2表示非2, 清零
        A2(m, n)
    }
    fi df&1 > 0 {
        df = df&-1
        A1(a, b)
    }
    return
}
```

官方称之为open coded defer

1.13和1.14中的defer都不适用于循环中的defer，循环中的defer需要使用1.12版本的defer，因此都会有`heap bool`的标记

但是1.14中的如果遇到panic，那么后面的defer将无法继续执行，又因为这样的open coded没有注册到链表，所有需要额外的栈扫描来发现。

1.14中defer更快了，但panic更慢了。但是defer发生的概率比panic大得多


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



