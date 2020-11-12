---
title: 制作一个简易的中间件架构
date: 2020-11-11
tags: middleware, tech, go
---

## 为何需要中间件

我们不应该把业务逻辑和非业务逻辑揉在一起。非业务逻辑如打印日志、计时等。因为如果我们需要一个新的日志系统，而我们打印日志的逻辑杂揉在每个handler中，那我们就得修改每个handler，费时费力且不明智。

中间件就是一种剥离非业务逻辑的方法。


## 原理

我们可以使用函数闭包来轻松实现剥离业务逻辑和非业务逻辑。

假设我们的业务就是以各种姿势处理字符串然后打印。如处理成`===str===`然后打印。

然后现在我们有个业务逻辑之外的要求，那就是打印完成后在结尾打印`done`。我们可以简单想到下面业务逻辑和非业务逻辑杂糅的代码。

```go
func helloHandler(s string){
    // 处理业务逻辑
    fmt.Println("==="+str+"===")
    // 处理非业务逻辑
    fmt.Println("done")
}
```

当然这个处理的逻辑还是比较简单的。如果我们有100个handler以不同姿势打印字符串，如`"str"`, `~str~`等，但是现在我们不说done了，要说ok。那我们就得为100个handler修改非业务逻辑的代码。单单这么一个简单的逻辑修改起来就挺费劲了。

那么我们现在用一种优雅的方式来分离业务逻辑和非业务逻辑， **函数闭包** 

```go
type HandleFunc func(s string)

func endPoint(h HandleFunc) HandleFunc{
    return func(s string){
        h(s)
        fmt.Println("done")
    }
}

func business(s string){
    fmt.Println("==="+s+"===")
}

func main(){
    f := endPoint(business)
    f("str")
}
```

我们成功将业务逻辑和非业务逻辑分离，当需要修改非业务逻辑时只需要专注修改非业务逻辑部分(`endPoint`)，而不需要对每个handler进行修改。

我们的中间件就是通过包装handler(进行些处理)，再返回一个handler，实际上就是不断地进行函数的压栈出栈。

**中间件吃什么吐什么**对用户是透明的

## 优雅的中间件写法

```go
type HandleFunc func(s string)

// 中间件就是一个吃什么吐什么的函数
type middleware func(HandleFunc) HandleFunc

type Router struct {
	middlewareChain []middleware
	route           map[string]HandleFunc
}

func New() *Router {
	return &Router{
		route: make(map[string]HandleFunc),
	}
}

func (r *Router) Use(m middleware) {
	r.middlewareChain = append(r.middlewareChain, m)
}

func (r *Router) Add(route string, h HandleFunc) {
    mergedHandler := h
	if len(r.middlewareChain) != 0 {
		for i := len(r.middlewareChain) - 1; i >= 0; i-- {
			mergedHandler = r.middlewareChain[i](mergedHandler)
		}
	}
	r.route[route] = mergedHandler

}

func cors(h HandleFunc) HandleFunc {
	return func(s string) {
		fmt.Println("set cors")
		h(s)
	}
}

func logger(h HandleFunc) HandleFunc {
	return func(s string) {
		fmt.Println("set logger")
		h(s)
	}
}
```

这样一来我们就可以通过`Use`来直观地增删中间件，并为对应的route设置handler了

```go
r := New()
r.Use(logger)
r.Use(cors)
r.Add("route1", func(s string){
    fmt.Println(s)
})
r.route["route1"]("business 1")
```

需要注意的是代码中`middlewareChain`的遍历顺序和`Use`的顺序相反，因为调用过程是`logger(cors(HanderFunc(s)))`，故从内到外包装



