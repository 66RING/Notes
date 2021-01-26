---
title: java注解与反射
date: 2020-7-17
tags: 
- java
- annotations
- reflection
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
- 正常方式
    * 引入需要的"包类(包名和类名)"名称 -> 通过new实例化 -> 取得实例化对象
- 反射方式
    * 实例化对象 -> `getClass()`方法 -> 得到完整的"包类"名称

- 优点
    * 可以实现动态创建和编译，灵活
- 缺点
    * 对性能有影响，使反射基本上是一种解释操作。比较慢


### Class类

`Class c1 = Class.forName("your.package.className");`


#### 获取Class类是实例

- 1. 已知具体类，通过类的class属性获取，改方法最为安全可靠，程序性能高
    * `Class clazz = Person.class;`
- 2. 已知某个类是实例，调用改实例的`getClass()`方法获取Class对象
    * `Class clazz = person.getClass();`
- 3. 已知一个类的全类名，且该类在类路径下，可通过Class类静态方法`forName()`获取，可能抛出ClassNotFoundException
    * `Class clazz = Class.forName("demo01.Student");`
- 4. 内置基本数据类型可直接用类名.Type
- 5. 可以利用ClassLoader，如获得父类的类型`son.getSuperclass();`


#### 获取类运行时结构

获取Class类实例后可以通过以下方法获取类运行时结构

- 获得属性
    * `v = obj.getFields();`
        + 可指定要获取的属性作为参数
        + 获取公有的属性
    * `v = obj.getDeclareFields();`
        + 可指定要获取的属性作为参数
        + 获取全部属性
    * 使用：set，`v.set(obj, value);`
- 获得方法
    * `m = obj.getMethods();`
        + 可指定要获取的方法(方法名和参数)作为参数
        + 获取本类和父类的公有的方法
    * `m = obj.getDeclareMethods();`
        + 可指定要获取的方法(方法名和参数)作为参数
        + 获取本类的全部方法
    * 使用：Invoke(激活)，`m.invoke(obj, args);`
- 获得构造器
    * `obj = clazz.getConstructors();`
        + 可指定要获取的构造器(需要的参数)作为参数
        + 获取公有的构造器
    * `obj = clazz.getDeclareConstructors();`
        + 可指定要获取的构造器(需要的参数)作为参数
        + 获取本类的全部构造器
    * 使用：newInstance，`Type a = (Type)obj.newInstance(args);`
- 获得泛型


#### 内存分析

- java内存
    * 堆
        + 存放new的对象和数组
        + 可以被所有的线程共享，不会存放别的对象引用
    * 栈
        + 存放基本变量
        + 引用对象的变量
    * 方法区
        + 可以被所有线程共享
        + 包含了所有的class和stack变量
- 类加载的过程
    * 1. 类加载(Load)
        + 将类的 **class文件** 读入内存，并为之创建一个java.lang.Class对象，此过程由类加载器完成
        + 加载：将class文件字节码内容加载到内存，并将这些静态数据转换成方法区的运行时数据结构，然后生成一个代表这个类的java.lang.Class对象
    * 2. 类的链接(Link)
        + 将类的二进制数据合并到JRE中
        + 连接：将java类的二进制代码合并到JVM的运行状态之中的过程
    * 3. 类的初始化(Initialize)
        + JVM负责对类进行初始化
- 类的初始化
    * 主动引用(一定发生类的初始化)
        + 当虚拟机启动，先初始化main方法所在的类
        + new一个类的对象
        + 调用类的静态成员和静态方法
        + 使用java.lang.reflect包的方法对类进行反射调用
        + 当初始化一个类，如果其父类没有被初始化，则先初始化它的父亲
    * 被动引用(不会发生类的初始化)
        + 当访问一个静态域时，只有真正声明这个域的类才会被初始化。如：当通过子类引用父类的静态变量，不会导致子类初始化
        + 通过数组定义类引用，不会触发类的初始化
        + 引用常量不会触发类的初始化


#### 类加载器的作用

加载器将class文件字节码内容加载到内存，并将这些静态数据转换成方法区的运行时数据结构，然后生成一个代表这个类的java.lang.Class对象，作为方法区中数据的访问入口。

类缓存：标准的JavaSE类加载器可以按要求查找类，一旦某个类被加载到类加载器中，它将维持加载(缓存)一段时间。JVM的垃圾回收机制可以回收这些Class对象。


## 反射操作注解

以ORM(Object relationship Mapping，对象关系映射)为例。类映射到数据库，类和表结构对应，属性和字段对应，对象和记录对应。

``` java
public class Students{
    public static void main(String[] args) throws ClassNotFoundException{
        // 反射创建对象
        Class c1 = Class.forName("pkg.of.yours.Student");

        // 通过反射获得注解
        Annotation[] annotations = c1.getAnnotations();
        for(Annotation annotation : annotations){
            System.out.println(annotation);
        }

        // 获得注解的value的值
        TableStudent tablestudent = (TableStudent)c1.getAnnotation(TableStudent(TableStudent.class));
        String value = tablestudent.value();
        System.out.println(value);
        
        // 获得类指定的注解
        Field f = c1.getDeclareField("name");
        // 反射回去对象的属性
        FieldStudent annotation = f.getAnnotation(FieldStudent.class);
        System.out.println(annotaion.columName());
        System.out.println(annotaion.len());

    }
}


@TableStudent("db_student")
class Student{
    @FieldStudent(columName = "db_age", len = 10)
    private age;
    @FieldStudent(columName = "db_name", len = 10)
    private name;
    // getter and setter and construction
}


// 自定义类上的注解
@Target(ElementType.TYPE) 
@Retention(RetentionPolicy.RUNTIME)
@interface TableStudent{
    String value();
}

// 自定义属性上的注解
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@interface FieldStudent{
    String columName();
    int len();
}
```

<++>









