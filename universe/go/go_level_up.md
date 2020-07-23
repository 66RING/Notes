---
title: Golang提高
date: 2020-7-23
tags: go
---

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



