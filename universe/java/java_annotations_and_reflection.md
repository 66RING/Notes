---
title: [java]注解与反射
date: 2020-7-17
tags: java, annotations, reflection
---


## 注解

注解可以让程序读取，格式`@注解名(参数)`

- 常用注解
    * `@Override`：重写的方法
    * `@Deprecated`：已经废弃的方法
    * `@SuppressWarnings`：镇压警告


### 元注解

**元注解负责注解其他注解** 。java定义了4个标准meta-annotation类型，他们被用来提供对其他annotation类型作说明

- `@Target`：表示这个注解可以注在什么地方：类、方法等
- `@Retention`：表示需要在什么级别保存注释信息，用于描述注解的生命周期(SOURCE < CLASS < RUNTIME)
- `@Documented`：说明改注解将被包含在javadoc中
- `@Inherited`：说明子类可以继承父类中的注解

### 定义注解

使用`@interface`自定义注解时，自动继承了`java.lang.annotation.Annotation`接口。

- `@interface`用来声明一个注解，格式`public @interface 注解名{定义内容}`
- 其中的每一个方法实际上是声明了一个配置参数
- 方法名就是参数的名称
- 返回值类型就是参数的类型(返回值只能是基本类型)
- 可以通过default来声明参数的默认值
- 如果只有一个参数成员，一般参数名为value，然后此时调用这个注解时可以省略一个value`@MyAnnotaion("hi")`
- 注解元素必须要有值，我们定义注解时，经常使用空字符串、0作为默认值
 
``` java
@interface MyAnnotation{
    String name();  // 是注解的参数，而非一个方法
    String addr() default "";   // 可设默认值 
}

@MyAnnotation(addr = "home", name = "ring")
public void Func(){ }
```


## 反射

- 反射让java具有动态性，反射机制允许程序在执行期间借助于Retention API获取任何类的内部信息，并能直接操作任意对象的内部属性和方法
- 加载完后，在堆内存的方法区中就产生了一个Class类型对象(一个类只有一个Class对象)，这个对象就包含了完整的类的结构信息。我们可以通过这个对象看到类的结构。这个对象就像一面镜子，透过这个镜子看到类的结构




