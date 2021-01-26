---
title: Golang gui 编程笔记
date: 2020-8-11
tags: 
- go
- gui
---


## 基本结构

- 初始化
- 用户设置
    * 1. 创建主窗口
        + `	win := gtk.NewWindow(gtk.WINDOW_TOPLEVEL)`
    * 2. 设置窗口属性
        + `	win.SetTitle("gtk go")`
    	+ `win.SetSizeRequest(400, 320)`
    * 3. 创建容器控件(固定布局、任意布局)
        + `layout := gtk.NewFixed()`
    * 4. 布局添加到窗口上
        + `win.Add(layout)`
    * 5. 显示控件
        + `win.ShowAll()`，否则一个一个show
- 主事件循环


### 控件

控件包含：属性、方法


### 信号处理

形如

- 注册按钮处理点击事件
    * `b1.Connect("clicked", handlefunc, args)`
    * 也可以使用封装好的`b.Clicked(func(){})`
    * `handlefunc(ctx *glib.CallbackContext)`进行处理，接受args参数
        + 也可使用匿名函数
    * 使用`ctx.Data()`获取传入的参数。返回空接口类型，可以使用断言
        + `data, ok := ctx.Data().(string)`，如果是string类型则`if ok通过`
- 关闭窗口触发destroy
    * `win.Connect("destroy", func(){gtk.MainQuit()}, args)`


## 使用glade辅助

- 编辑好后加载glade文件：
    * `builder := gtk.NewBuilder()`
    * `builder.AddFromFile("xxx.glade")`
- 获取glade上的控件
    * `win := gtk.WindowFromObject(builder.GetObject("yourObjName"))`，获取窗口
    * 显示控件，如果是glade添加的控件，`show`即可显示所有
        + `win.Show()`
    * 信号处理的方法页一样

