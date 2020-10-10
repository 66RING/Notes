---
title: Tensorflow   
date: 2019-9-19   
tags: python, tensorflow, machine learning
mathjax: true
---



# 基本操作

### 图片读取展示  
```python
import cv2  # 引入OpenCV
img = cv2.imread('path',1)  # 读取图片，0是灰图，1是彩图
cv2.imshow('image',img)  # 'image'打开的窗体的标题，img展示的内容
cv2.waitKey(0)  # 暂停
```
cv.imread 过程：1文件读取 2封装格式解析 3数据解码 4数据加载  

### 读写操作  
#### 图片读写  
```python
import cv2
img = cv2.imread('path',1)  # 读取图片，0是灰图，1是彩图
cv2.imwrite("path",img)  # 1,图片名 2.图片数据

# 不同质量的图片写入
# jpg,有损压缩
# 压缩比参数范围为0~100，越低压缩比越高
cv2.imwrite("path.jpg",img,[cv2.IMWRITE_JPEG_QUALITY,0]) 

# png是无损压缩，有透明度属性
# 压缩比参数0~9,越低压缩比越低
cv2.imwrite("path.png",img,[cv2.IMWRITE_PNG_QUALITY,0]) 
```

#### 操作像素
```python
import cv2
img = cv2.imread("img.jpg",1)
# OpenCv读取图片是bgr(rgb倒过来)，左上角开始的坐标轴

# 读取像素点
(b,g,r) = img[x,y]
print(b,g,r)

# 写入像素
img[x,y] = (b,g,r)
```





---  
# OpenCv  

### OpenCv模块结构  
```
to be continued
```

--- 


# Tensotflow  

#### 基本操作 
##### 概况
```python
import tensorflow as tf
# 定义常量
data1 = tf.constant(2.5)  # 指定数据类型可以加参数(2,dtype=tf.int32)
# 定义变量
data2 = tf.Variable(b,name="name")
# 打印出来的是描述信息


# 所有操作要session会话进行
sess = tf.Session()
print(sess.run(data1))  # 通过会话进行的就可以打印了

# 所有变量都要用session进行初始化
init = tf.global_variables_initializer()
sess.run(init)  # 初始化
# session打印多个内容
sess.run([x1,x2,x3])


# 关闭session
# 法一
sess.close()
# 法二  with
with sees:
	...
```

##### 类型
```python
# tensorflow运算的每个类型都要是tensor
# 转换为tensor,如 a=np.arange(1)
aa = tf.convert_to_tensor(a,dtye=tf.int32) #dtype=数据类型

# tensor类型间转换
tf.cast(aa,dtype=tf.double)


# Variable
# Variable包装过的变量会具有一些特殊的属性,如可导
b=tf.Variable(b,name="name")
b.name
b.trainable

# tensor变numpy
# tensor一般在GPU,当有时我们要在CPU上处理默写逻辑时就要转成numpy
a.numpy()  # tensor:a 就变成了numpy
```

##### 创建tensor
```python
a.convert_to_tensor()

# 初始化
tf.zeros(shape)  # tf.zeros_like(a) 初始化一个和a一样维度的(shape)
tf,ones(shape)
tf.fill(shape,elem)
tf.random.normal(shape,mean=1,stddev=1)  # 用正态分布采样(normal,其他分部同理)初始化一个,其中mean,stddev正太分部的参数,其他分部同理
tf.random.truncated_normal(...)  # 截断的正态分布
tf.random.uniform(shape,minval=0,maxval=1)  # 均匀分布采样
# shape表示维度



# 随机打散
# 就是random了,但是如果是两组有一一对应关系的东西,怎么打散才不会破坏那个一一对应关系?
idx = tf.range(10)  # 假设有10组数据
idx = rf.random.shuffle(idx)  # (就好比生成了10组随机的通道(每个通道代表一种一一对应关系)通道两边绑定了,所以对应关系不变)



```

##### 索引与切片  

```python 
# 索引
# numpy风格的索引，如：
a.shape() = [1,2,3,4]
a[1,2].shape = [3,4]
# 索引写在一个[]内，用逗号隔开



# 切片
# 对于某个维度
a[-1:]  # 到数第一个到最后一个,就是python的切片
# 对多个维度的切片
a[0,1,:,1:3,:]  # (取a01的全部的1到3的全部。。。很灵活)
# step,步长.... [::] 同理, 步长为负，实现倒叙

# 省略号:省略多个:(自动识别)
a[1,2,...,0,:]  # 中间的没有切片操作,但是倒数第二有切片操作,用...就不用人为的把中间的:不上了



# Selective Indexing 
# 可以乱序取样
# 假设a.shape = [4,32,8] ,a[4个班,35个学生,8门科目成绩]
a = tf.gather(a,axis=0,indices=[2,1,3,0])
# 1.取样的样本 2.抽取的维度,上面就是从第一个维度中乱序的抽取,随机抽取一个班查看 3.抽取的顺序,抽2班1班3班0班

# 还是上面的例子,如果想要取n个学生的m门成绩呢？
aa = tf.gather(a,axis=1,indices=[2,1,3,0])  # 取4个班2，1，3，0号学生
aaa = tf.gather(aa,axis=2,indices=[2,1,3,0])  # 取这4个班2，1，3，0号学生,的2，1，3，0号成绩
# 多个gather嵌套

# tf.gather_nd !!!比较难理解
gather_nd(a,[1,1,1])  # 1班1号同学的1号成绩,标量
gather_nd(a,[[0,0],[1,1]])  # 0班0号同学的8门成绩,和1班1号同学的8门成绩,组成的矩阵,shape = [2,8] 2个同学,8门成绩
gather_nd(a,[[0,0,1],[1,2,1]])  # shape = [2]
gather_nd(a,[[[0,0,1],[1,2,1]]])  # shape = [1,2]
# 体会标量放[]里和矩阵放[]里的区别,差不多就是这个意思

# tf.boolean_mask
# 通过boolean来取样 假设a.shape = [4,28,28,3]
tf.boolean_mask(a,mask=[True,True,False,False])  # 默认从最外层(mask嘛)
# 结果shape = [2,28,28,3]
# 多维遮罩 例:a.shape = [2,3,4]
tf.boolean_mask(a,mask=[[True,False,False],[False,True,True]])  # mask.shape=[2,3] 采样的元素取对应关系,根据mask,第0行第一个元素是True，所以要...
# 结果shape = [3,4]

# 当然也可以指定,遮罩哪个维度 
tf.boolean_mask(a,mask=[True,True,False],axis = 3)  # shape = [4,28,28,2]
```

##### 维度变换
```python
# a.shape = [4,28,28,3]
tf.reshape(a,[4,784,3])  # 4*28*28*3  ==  4*784*3 才能保证所有数据充分利用
# 如果先偷懒的话可以用-1
tf.reshape(a,[4,-1,3])  # 一个式子只能有一个-1,-1就相当于x,保证4*28*28*3 == 4*x*3
# 变换前要理清楚物理含义

# 矩阵变换,改变格式
# a.shape = [4,3,2,1] 
tf.transpose(a)  # 矩阵转置
tf.transpose(a,perm=[0,1,3,2])  # 原来的0维放在新的0维...原来的3维放在新的2维...
# 结果 shape = [4,3,1,2]

# 维度的增加
# a.shape=[4,35,8]
tf.expand_dims(a,axis=0)  # 插入的(一个)维度相当于插入后维度的第0维,a.shape=[1,4,35,8]
tf.expand_dims(a,axis=3)  # 插入的维度相当于插入后维度的第3维,a.shape=[4,35,8,1]
# 负数同理

# 维度减少
# 元素个数为1的维度是可以去掉的,a.shape=[1,2,1,1,3]
tf.squeeze(a)  # 不加axis参数就是把所有1去掉
tf.squeeze(a,axis=2)  # 把第二维度去掉
```

##### Broadcasting
- expand without copying data:扩张了一个数据,但实际上并没有复制出来多份
<img src="./static/broadcasting.png" style="zoom:50%">

```python
tf.broadcast_to
# ape=[3,5]
aa = broadcast_to(a,[4,3,5])
aa.shape = [4,3,5]


# 如前面的 x@w+b,b是一个一维的，但却能加上去,就是broadcast的功劳
# 如a.shape=[4,16,16,32] b.shape=[32]
# 如果a+b 那么b就会相当于自动变成[4,16,16,32],以满足相应的运算(包括加减乘除
# 先从小维度开始匹配,自动扩张是满足运算
# 但却不会生成4*16*16个b
# 判断方法:右对其,用1把维度补相同,然后把1是维度变成和另一个匹配的


# a.shape=[1,3,4]
# tf.tile(a,[2,1,3]) 第一个维度复制2ci，第二个1次，第三个4次
# a.shape = [2,3,12]
```

##### 数学运算
```python
# element-wise: +-*/
# shape一样，对应元素运算
# (一般的运算,非矩阵...吧)


# matrix-wise: @,matmul
# 如 [b,3,4]@[b,4,5]
# 相当于把后两个当成矩阵然后来运算[3,4]*[4,5] = [3,5]
# 相当于一下子b个矩阵相乘
# (矩阵运算...)


# dim-wise: reduce_mean/max/min/sum
```

##### 手写数字识别,你可能用到
```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import datasets

(xs, ys),_ = datasets.mnist.load_data()
xs = tf.convert_to_tensor(xs, dtype=tf.float32) / 255.    # 除以255是为了优化,这样0<x<1
ys = ....  # 变成tensor

train_db = tf.data.Dataset.from_tenfor_slices((x,y)).batch(128)
train_iter = iter(train_db)
sample = next(train_iter)


w1 = tf.Variable(tf.random.truncated_normal([784,256]),stddev=0.1)  # stddev=0.1是为了......
b1 = tf.Variable(tf.zeros([256]))   # 变成tf.Variable才能被Gradient跟踪
...
...
for epoch in range(10):   # 对整个数据集循环,反复使用用一个数据集不断优化
	for step,(x, y) in enumerate(train_db):  # step,方便记录,查enumerate用法
		x = tf.reshape(x,[-1,28*28])
		
		with tf.GradientTape() as tape:  # 默认只会跟踪tf.Variable的类型
			h1 = x@w1 + b1
			h1 = tf.nn.relu(h1)
			...
			out = ...
			
			y_onehot = tf.one_hot(y, depth=10)  # y:[b] => [b,10]
		
			loss = tf.square(y_onehot - out)
			loss = tf.reduce_mean(loss)
			
		grads = tape.gradient(loss,[w1,b1,w2,b2,w3,b3])
		# 更新w,b
		w1.assign_sub(lr*grads[0])  # 原地更新,引用不变,类型不变
		...
		...
		if step % 100 ==0:
			print(float(loss))


```

##### 合并与拼接
```python 
c =tf.concat([a,b],axis=0)   # a和b第0维度合并
# 在原有维度上累加,不能生成新的维度

# 如果要创造新的维度
# a.shape = [4,3,5] b.shape = [4,3,5]
c = tf.stack([a,b]axis = 1)
# c.shape = [4,2,3,5]  
# 根据表示意义理解 如[chool,class,student,scores]
### 以上对维度都有要求,有一定的局限性
# 同样用[class,student,scores] 模型举例
# 每个学校，班等都可能不同,stack就操作不了

tf.unstack(a,axis=0)   # 全部拆开,返回几个tensor取决于有几个
tf.unstack(a,axis=3,num_or_size_splits=2)   # 在指定维度拆开拆开,参数是2所以拆成两个
tf.unstack(a,axis=3,num_or_size_splits=[2,2,3])   # 指定拆开,拆开的低0个有2份,地2个有3份...
```

##### 数据统计

- 范数
  - 二范数  
	$${||x||}_2 = [\sum_k{x^2_k}]^\frac{1}{2}$$  
  - 无穷范数  
  - 一范数..等等
	$${||x||}_1 = \sum_k{|x_k|}$$  

```python
###  这里讨论的都是向量的范数(非矩阵)
tf.norm(a)  # 二范数
tf.norm(a,ord=1,axis=1)  # 一范数,同时把某维度看做整体来做范数


# reduce_mean/min/max
# reduce说明,这操作会有个减维的过程:相当于每组选出了指定的数,那组的大小就成了1
tf.reduce_mean(a,axis=1)  # 2.不指定维度的话会打平成以维度
# 指定了维度就会在指定维度取  $注意,这里讨论的都是向量,不用矩阵来理解

# 就最大最小值的位置
# a.shape = [4,10]
tf.argmax(a)  # 默认第0维比较,a有10组,所以会返回10个结果[2,3,4..]
tf.argmin(a,axis=1)  # 指定维度

# 比较
tf.equal(a,b) # 返回[True,False,True,...]
# 准确度:把上面的返回结果dtype成0,1然后累加

# tf.unique
tf,unique(a)
# 返回两个值,第一个是无重复值的tensor,第二个是tensor是值表示原tensor的元素在新tensor中的位置
# 这么一来可以用tf.gather来吧原tensor还原出来

```

##### 张量排序
```python
tf.sort(a,direction='DESCENDING')  # 降序,  direction='ASCENDING'就能升序
tf.argsort(a,direction='DESCENDING')  # 降序,返回的是位置:如[最大值位置，次大..]
# 同理可与gather配合
# 高维的话就按每维排列完全排序

# 但有时候我们只需要最大最小(不用完全排序,耗时)
res = tf.max.top_k(a,2)  # 返回最大的两个
res.indices   # 返回索引值,像argsort
res.values  # 返回值

# 应用
# 预测问题:0,1,2,3的预测分别是 prob[0.1,0.2,0.3,0.4]
# 真实值是2
# top-1 prediction(正确答案在前1个的概率):0%   (预测对的样本个数/总样本数(这了只用应该样本)) 
# top-2 prediction(正确答案在前2个的概率):100% 
# top-3 prediction(正确答案在前3个的概率):100% 
# 举例
# prob = tf.constant([[0.1,0.2,0.7],[0.2,0.7,0.1]]) #样本1最可能是2,样本2最可能是1
# target = tf.constant([2,0])  # 样本1正式值应该是2，样本2真实值应该是0
# 所以: top-1 prediction=1/2  = 50%
# top-2 prediction = 2/2 = 100%
# top-3 prediction = 2/2 = 100%
```

##### 填充与复制
```python
# 填充 pad
tf.pad(a,[[2,0],[0,1]])  # 行上边边填充2行下边0行;列左0右1
#          ^行  ^列  

# 复制 tile
# a.shape = [3,3]
tf.tile(a,[1,2])  # 第一维复制一次(不变),第二维复制2次 
# res.shape = [3,6]
# 会真实的复制到内存
```

##### 张量的限幅
```python
# 限制最小值
tf.maximum(a,2)  # 返回a,2间的最大值,故a不会小于2,限制的最小值
# 限制最大值
tf.minimum(a,8)  # 返回a,8间的最小值 

# 限制范围
tf.clip_by_value(a,2,8)  # 2<x<8

# relu函数,x小于0时取0，大于0是取本身
# 可用maximum(a,0)实现
# 也可用封装好的relu函数
tf.relu(a)

# 等比例放缩,希望把grad缩小方便学习,但又不希望改变gred值
# 可用除以模再乘以一个值来控制范围来,也可用函数
tf.clipe_by_norm(a,15)  # 相当于除模后乘15,改变了a的模



# Gradient Exploding 梯度太大,一步学习跨越太大,来回震荡
# Gradient Vanishing 梯度太小,学习太慢，长时间没有变化
# tf.clipe_by_global_norm(grads,25)  # 整体缩放,避免方向改变
# 梯度向量表示[2,5,3],那么整体缩小就不会改变方向
```

##### 高级操作
```python
# 筛选 mask = [True,False,True]
tf.where(mask)  # 没有参数,返回tensor中值是True的值的对应坐标tensor
tf.where(mask,A,B)  # True时对A采样,False时对B采样



# 有目的性的更新
tf.scatter_nd(indices,updates,shape) 
# 1.只能在全0的底板上更新,就是上面的shape
# 2.indices表示要更新的位置,把对应位置上updates的值更新过去
# 一般用作给指定位置加减(因为只能全0为底板)


# 快速生成坐标轴系(GPU加速的,区别于传统for循环的)
point_x,point_y = tf.meshgrid(x,y)
# 返回两个值,个存取x的所有值和y的所有值
# 对应位置的祝贺就是(x,y) 
# 重新组合: tf.stack([point_x,point_y],axis=2)
```

#### 神经网络与全连接层

##### 数据集的加载(小型)
```python
# 数据集准备
(x,y),(x_test,y_test) = keras.datasets.mnist.load_data()  # 获取mninst数据集,返回各有不同
# 返回的是numpy的格式

# 将numpy转换成对象
db = tf.data.Dataset.from_tenfor_slices(x_test,y_test)
next(iter(db))[0].shape  # 转换成对象后就可进行的一系列操作,支持多线程等

# 打散
db = db.shuffle(10000)  # 打散,但x和y的对应关系不打撒(gather),参数?给大点就是了


# 数据预处理
db2 = db.map(func)  # 对db里的每个元素进行func里的操作
# 如每个元素是(x,y),func函数的参数的x,y返回的是处理后的x,y

# batch
db3 = db2.batch(42)  # 不再一次读取一组数据,一次读取指定数量的数据

# 重复迭代
db4 = db3.repeat(2)  # 重复迭代2次
db4 = db3.repeat()  # 无限重复

```

##### 全连接层
```python
# 每个节点跟每个节点连接——Dense
x = tf.random.normal([4,728])  # 输入
net = tf.keras.layers.Dense(512)  # 创建输出512的层
out = net(x)  # out.shape = [4,512]


# 多层嵌套——Multi-Layers
# keras.Sequential([layer1,layer2,...])  # layer->Dense
network = keras.Sequential([
        keras.layers.Dense(2,activation='relu'),
        keras.layers.Dense(2,activation='relu'),
        keras.layers.Dense(2)
    ])
network.build(input_shape=[None,3])  # 创建，给定输入维度3
network.summary()   # 打印信息
network.trainable_variables   # list[],可训练参数
```

##### 输出方式
```python
# 输出范围压缩
# sigmod函数(同理relu)
y = tf.sigmod(x)   # x属于R,y属于[0,1]

# tanh函数,压缩范围到[-1,1]
tf.tanh(x)


# 输出概率(所有的和为1)
# softmax函数
tf.softmax(a)
```

##### 损失函数的计算
- MSE  
$$loss=\frac{1}{N}\sum(y-out)^2$$
```python
loss1 = tf.reduce_mean(tf.square(y-out))
loss2 = tf.reduce_mean(tf.losses.MSE(y,out))
# loss1 = loss2 等价
```

##### 标差熵
- 熵 $Entropy = -\sum P(i)\log_2{P(i)}$
    - 不确定度 Uncertainty
    - 惊奇度 measure of surprise
    - lower entropy -> more info


##### 交叉熵 Cross Entropy
- 描述两个集合p,q的惊奇度
    - $H(p,q) = -\sum{p(x) \log_2{q(x)}}$
    - $H(p,g) = H(p) + D(p|q)$ 
        - $D(p|q)$ 表示p和q的离散度
        - 当p=q时$D(p|q)=0$

- for p:one_hot encoding
    - $h(p:[0,1,0]) = -1\log_2{1}=0$
    - $H([0,1,0],[q_1,q_2,q_3]) = 0+D(p|q)=-1\log{q_1}$
    - 即要使p逼近与q用交叉熵的方法的可行的  

- 具体解法
设一组分类的one_hot encoding是$P_1[1,0,0,0,0]$;  
一组输出为$Q_1[0.4,0.3,0.05,0.05,0.5]$;  
则:

$$\begin{aligned}
loss &= H(p,q) \\
&= -\sum{P_1(x) \log_2{Q_1(x)}} \\
&= -\log_2{0.4}  \\
&= 0.916
\end{aligned}
$$


然后lr,w1,b2...,多次学习后发现loss越来越小,即q = p  
- 在tensorflow中的使用

```python 
tf.losses.categorical_crossentropy(p,q) # 函数的形式
tf.losses.BinaryCrossentropy()(p,q)  # 类的形式
tf.losses.binary_crossentropy(p,q) # 函数的形式 
# p是真实在的one_hot encodingq是预测值
# 如tf.losses.categorical_crossentropy([1,0,0,0],[0.25,0.25,0.25,0.25])


### 通常的用法 ###
tf.losses.categorical_crossentropy(one_hot,logits,from_logits=True)  # 这样能处理logits转换成prob时的错误
tf.losses.categorical_crossentropy(one_hot,prob)  # 等价但不推荐
```


### 梯度下降 Gradient Descent

- 梯度:向量grad
    - 用梯度下降来逼近
$$ w_n = w - lr \times \frac{\partial{loss}}{\partial{w}} $$
- 在tensorflow中的使用

```python 
with tf.GradientTape() as tape:  # 把计算过程包在里面
    tape.watch([w,b])  # 如果参数不是tf.variable类型话要用这个函数声明
    loss = f(x)
[w_grad] = tape.gradient(loss,[w])  # 自动求解参数的梯度,并返回相应的列表

# tape.gradient调用一次后会把资源释放掉,可用参数persistent改变
with tf.GradientTape(persistent=True) as tape:  # 用完后会保留资源
grad1 = tape.gradient(loss,[w]) 
grad2 = tape.gradient(loss,[w])  # 可调用多次
# 但要记得手动释放资源！！！
```


#### 激活函数 Activation Function
科学家在研究青蛙神经是发现，当刺激到达一定程度是青蛙才会做出相应的反应，是个离散的过程  
因此在深度学习中就可模仿设点，设计神经网络，因此有了激活函数  


连续的光滑的激活函数
- **sigmoid(logistic)**
    - $f(x)=\delta(x)=\frac{1}{1+e^{-x}}$
    - `y = tf.sigmoid(a)`
    - 可以将范围压缩到[0,1]
    - 但当x接近无穷时，导数几乎为零，导致梯度离散，使得长期得不到更新
- **Tanh**
    - $f(x)=tanh(x)=\frac{(e^x-e^{-x})}{e^x+e^{-x}}=2sigmoid(2x)-1$
    - `y = tf.tanh(a)`
- **ReLU(Rectified Linear Unit)**
    -   $
        f(x) = \begin{cases}
        0, & \text{if } x < 0  \\
        x, & \text{if } x \geq 0 
        \end{cases}
        $
    - `tf.nn.relu()`
    - 深度学习最常用的
        - 优势
        - 求导简单
        - 不会放大或缩小梯度(reLU的导数为1)
- **Softmax**
    - $S(y_i)=\frac{e^{y_i}}{\sum_j{e^{y_i}}}$
    - 常用于多分类问题，因为它把logits转换为prob
    - 区别于一般的转换成prob的方法，Softmax会把大的放大，小的缩小；拉大差距(sotf version of max)
    - 求导:把先把分子分母看做整体`f(x)和g(x)`然后相当于$\frac{\partial p_i}{\partial a_j}=\frac{f'(x)g(x)-f(x)g'(x)}{g(x)^2}$;注意i和j不同的情况要分开讨论
        - $$
            \frac{\partial p_i}{\partial a_j} = \begin{cases}
            p_i(1-p_1), & \text{if } i=j  \\
            -p_jp_i, & \text{if } i\neq j 
            \end{cases}
          $$



#### Loss函数的梯度

经典的loss函数
- Mean Squared Error(MSE,均方差)
    - $loss=\frac{1}{N}\sum(y-out)^2$
    - `loss1 = tf.reduce_mean(tf.square(y-out))`
    - `loss2 = tf.reduce_mean(tf.losses.MSE(y,out))`
- Cross Entropy Loss
    - Softmax


#### 链式法则

$\frac{\partial y}{\partial x} = \frac{\partial y}{\partial u}\frac{\partial u}{\partial x}$



#### 感知机梯度传导
利用链式法则从输出往输入退就可以知道梯度信息，然后更新  


### 可视化
- tensorboard
    - `pip install tensorboard`
    - 在代码中写入`summary_writer = tf.summary.create_file_writer(DIR)`
    - 拿到`summary_writer`后就可以忘里面喂数据

```python
# 1,喂数据点
with summary_writer.as_default():
    tf.summary.scalar('NAME1', float(LOSS), step=STEP)  # (图的名字,数据,坐标(默认是x轴))

# 2,喂一个图片
with summary_writer.as_default():
    tf.summary.image('NAME1', IMG, step=STEP)  # (图的名字,数据,坐标(默认是x轴))

# 3,给多个图片
# 最好的办法是认为的拼接图片,然后传一张拼接的图片(google)
```

- visdom 



## Keras高层API

#### 优化
在计算loss,accuracy的时候经常会发现数据忽高忽低,所以可借助keras的api来优化

- metrics测量
    - keras会将数据放在一个list,然后取平均值来优化?
    - 如`loss_meter = metrics.Mean()`,`acc_meter = metrics.Accuracy()`
- update_state更新数据
    - `loss_meter.update_state(loss)`,`acc_meter.update_state(y, pred)`
- result().numpy()获取结果,转换成numpy输出
    - `loss_meter.result().numpy()`result得到tensor，再转换成numpy
- reset_states释放数据
    - 当要废弃旧的数据时`loss_meter.reset_states()`

#### Compile&Fit

- Compile,类似装载弹药,可以指定loss,优化器,评估指标
- Fix,完成标准创立
- Evaluate,测试
- Predic,拿创建好的模型来预测

```python
### 一般的流程
epoch in range(num):
    for step, (x, y) in enumerate(db):
        with tf.GradientTape() as tape:   # 循环网络
            # [b, 784] => [b, 10]
            logits = model(x)
            y_onehot = tf.one_hot(y, depth=10)

            loss_ce = tf.losses.categorical_crossentropy(y_onehot, logits, from_logits=True)
            loss_ce = tf.reduce_mean(loss_ce)

        grads = tape.gradient(loss_ce, model.trainable_variables)    # 更新
        optimizer.apply_gradients(zip(grads, model.trainable_variables))

        if step % 100 == 0:   
            # ...
    for (x_test, y_test) in test_db:    # 测试


### 使用Keras的api快速建立标准化的神经网络
# 称network或model
network = Sequential([...])   # 如果是别的没学到的话...
network.compile(
        optimizer=optimizers.Adam(lr=0.01),    # 指定优化器
        loss=tf.loss.CategoricalCrossentropy(from_logits=True),   # 指定loss函数
        metrics=['accuracy']     # 指定测试标准
    )

network.fit(
        db,   # 要训练的数据集
        epochs=10,    # 训练的周期
        validation_data=db_test,    # 用于做测试的数据集,一般写作ds_val
        validation_freq=2    # 测试的周期,如这里一共10个epochs,每2个epochs就进行一次测试
    )

network.evaluate(ds_val)    # 训练完后对模型的评估,传入一个数据集

pred = network(x)
# 或 pred = network.predict(x)    预测
```


#### 自定义网络

- keras.Sequential(layer1, layer2, ...)
    - 参数要继承自`keras.layers.Layer()`
    - 建立好网络后variable(w和b)是没有的
        - 法1:指定输入shape`network.build(input_shape=(None, 28*28))`
        - 法2:自动识别`network(x)`
            - 这个的原理是调用了类中的call()方法,相当于network.__call__(x)。同理自定义类中也可如此
- keras.layers.Layer()
    - 任何要自定义的层要继承自它
- keras.Model()
    - compile/fit/evaluate
    - Sequential也是继承自该类，所以自定义的网络应该继承这个

```python
class MyDense(layers.Layer):    # 自定义层继承
    
    def __init__(self, inp_dim, outp_dim):
        super(MyDense, self).__init__() 
        self.kernel = self.add_weight('name1', [inp_dim, outp_dim])   # 用母类的add_weight而不是用tf.variable
        self.bias = self.add_weight('name2', [outp_dim])    # name是给母类管理用的
        
    def call(self, inputs, training=None):
        out = inputs @ self.kernel + self.bias
        return out

# 对比
layers.Dense(256, activation=tf.nn.relu),

# 同理Model自定义方法也一样
```

#### 模型的加载与保持

- save/load weights
    - 只保存模型参数
    - 缺点是没有源代码，网络不得而知
- save/load entire model
    - 简单粗暴的
- saved_model 
    - 通用的保存格式
    
**save/load weights**

```python
# save
model.save_weights('PATH')

# load
model = create_model()    # 需要人工创建网络
model.load_weights('PATH')
```

**save/load entire model**

```python
# save
model.save('PATH')

# load
model = tf.keras.models.load_model('PATH')  # 不需要人工创建网络
```

**saved model**

```python
# save
tf.saved_model.saved(model, 'PATH')   # 标准的，可供其他模型使用的保存

# load
imported = tf.saved_model.load(path)   

# 还原除网络
f = imported.signature['serving_defaut']
```



### 过拟合和欠拟合
现实情况是我们并不知道模型的符合什么分布  

- model capacity,模型的学习能力
    - 显然项越多越高
- underfitting
    - 模型的表达能力弱于真实数据，如用直线拟合双曲线
- overfitting
    - 模型的表达能力大于真实数据，把不必要的噪声也拟合进来了
    - 最常见

#### 检查overfitting
##### 交叉验证
检查欠拟合和过拟合的方法   

一般情况下会把数据集切分(splitting)成三份,作用分别是train set，val set，test set   
数据集一部分用来训练，一部分用来验证accuracy这是是显然的，那为什么有第三份呢？  
因为在真实的需求中，是不是有取巧的人会把test用的数据集也用来训练，从而过拟合来达到很高的准确度(但实际它们已经过拟合了)  
所以第三份是用来防止这种情况发生的，不参与训练的，最终检验模型的数据集


```python
network.compile(
        optimizer=optimizers.Adam(lr=0.01),   
        loss=tf.loss.CategoricalCrossentropy(from_logits=True),   
        metrics=['accuracy']   
    )

network.fit(
        db,    # training
        epochs=10,
        validation_data=db_test,   # val set
        validation_freq=2   
    )

network.evaluate(ds_val)   # test set
```

##### K-fold cross-validation
由上面知，test set是完全不能动的，所以在切分的时候train set和val set可以随机的切分，可以防止网络记忆特性

```python
# 在tensorflow中可以表现为
shuffle(db)  # 打散
splices()   # 切割

# 也可用keras的功能
network.fit(db, validation_split=0.1)   # 按照9:1随机切分
```

#### 减轻overfitting
##### Regularization

- L1-regularization
    - loss加上lambda约束的一范式
    - $j(\theta) = -\sum^m_1{y_i\log_e{\bar y_i} + (1-y_i)\log_e{(1-\bar y_i)}} + \lambda \sum_i^n{|\theta_i|}$
- L2-regularization
    - loss加上lambda约束的一范式
    - $J(W;x,y)+\frac{1}{2} \times ||W||^2$

```python
# 法一：在一层网络中添加kernel_regularizer参数
keras.layers.Dense(16,
                    kernel_regularizer=keras.regularizers.L2(0.001)   # 0.001就是 lambda
            )


# 法二：更加灵活的自己控制范式
loss_regularization = []   
for p in network.trainable_variables:     # 取范式里面的参数w1,w2...b1,b2...取法很灵活
    loss_regularization.append(tf.nn.l2_loss(p))
loss_regularization = tf.reduce_sum(tf.stack(loss_regularization))  # 做一范式还是二范数...

loss = loss + 0.0001*loss_regularization
```

#### 动量与学习率
##### Momentum 动量
由于梯度的更新，会有大幅的反复跳跃的现象，动量就是在更新方向的基础上结合上一阶段的方向进行梯度更新，从而使得更平缓，像踩刹车一样

```python
optimizer = SGD(learing_rate=0.02, momentum=0.9)   # momentum 就在超参数lambda
optimizer = RMSprop(learing_rate=0.02, momentum=0.9)
optimizer = Adam(learing_rate=0.02,   # Adam没有momentum(内置),但有beta_1,beta_2
        beta_1=0.9,
        beta_2=0.999) 

```

##### Learning rate 学习率
学习率动态调整来优化网络

```python
optimizer = SGD(learing_rate=0.02)
for epoch in range(100):
    # get loss

    # change learing_rate 比较简单粗暴
    optimizer.learing_rate = 0.2*(100-epoch)/100

    # update weights
```

#### Early Stopping & Dropout
##### Early Stopping
很多情况下虽然training accuracy还在上升，但是validation accuracy以及达到最优甚至开始下降了，这是就需要以前终止

##### Dropout
和overfitting的情况一样，为减少噪声的干扰，可以减少节点数(?矩阵里面的?),learning less to learning better

```python
network = Sequential([layers.Dense(256, activation='relu'),
                      layers.Dropout(0.5),    # 0.5 rate to dropout
                      layers.Dense(256, activation='relu'),
                      layers.Dropout(0.5),    # 0.5 rate to dropout
                      ...
                    ]) 
```
因为training和test的策略不同(training时为得到更好的w,b，而使用dropout的方法来减小overfitting,所以开启dropout，test是测试模型，所以不用开)

```python
# training
network(x, training=True)

# validation || test
network(x, training=False)
```

##### Stochastic
##### Deterministic


## 卷积神经网络
在处理图像问题时，使用全连接的方式会导致大量的资源占用.  
于是由生物学上眼睛可视域的启发，我们采用局部连接，然后滑动直至扫描全部输入。特点在于对于相同的层如(RGB),每次扫描的观察方式(卷积核)是一样的(weight sharing)  
所以学习的时候就大大减少了参数量  

### 卷积
信号的叠加叫做卷积,得到的结果叫做**feature map**  

$$
y(t)=x(t) * h(t)=\int^\infty _ {-\infty}  x(\tau)h(t-\tau)\mathrm{d}x
$$

\* 表示卷积操作,x就相当于输入,h就相当于观察方式(卷积核),t就相当偏移量，扫过整个图片t发生改变x和h卷积出信号输出y

#### Padding & Stride
- Padding
    - 把输入层扩大(虚的)然后扫描后就能得到维度与输入相等的输出
- Stride
    - 把扫描的步长加大，就能减少输出的维度

```python
layers.Conv2D(4, kernel_size=5, stride=1, padding='samd')  # 卷积核个数,5*5,步长,'same'可以保证输入维度等于输出
```

#### Channels

- 设输入是[1, 32, 32, 3],32\*32的图片,3个通道
    - 那我们的一个卷积核可以是[3, 5, 5] 3表示输入通道的数量(RGB)
    - 最后可以得到一个[b, 30, 30, 1]的输出
- 如果使用多个核如[N, 3, 5, 5]那就能得到N个[b, 30, 30, 1]即[b, 30, 30, N]

多通道输出，多通道输入

#### Gradient

$$
O _ {mn} = \sum {x _ {ij} * w _ {ij}} + b  \\
\frac{\delta Loss}{\delta w _ {ij}} 
$$


### Classic Network

#### GoogLeNet

When the network get deeper, above 20, is get harder to training, even make trains revoke.


#### ResNet

Residual


## Sequence
Signal with time order

- sequence embed
    - turn digital signal into a sequence

Many sets can be like a sequence. mnist for example[b, 28, 28]. can expand like [b, time, 28] or [time, b, 28] and so on.

But a sequence better to expand like a time orde things [time, b, 28] is much better. It depend on how you expand.

Here are some rules:
- semantic similarity
- trainable

### Cycle network
Two question:
- Long sentence
    - weight sharing
    - We can do like a conv_net

- Context information
    - It is a pertinence bettween word and word
    - Here is the example formulation

$$\begin{aligned}
h_t &= f_w(h_{t-1}, x_t) \\
h_t &= tanh(W_{hh}h_{t-1} + W{xh}x_t) \\
y_t &= W_{hy}h_t \\
\end{aligned}$$


### RNNlayer

#### SimpleRNN
$$
\begin{aligned}
call &= xw_{xh} + h_tw_{hh}, (for\ each\ item\ in\ timeline) \\
out_1, h_1 &= call(x, h_0) \\
out_2, h_2 &= call(x, h_1) \\
out_t, h_t &= call(x, h_{t-1}) 
\end{aligned}
$$

$h_t$ and $out_t$ is the same thing(id) but have difference meaning 


#### Optimize
- Step 1:Gradient Exploding
    - Gradient Clipping
    - $grad = \frac{|grad|}{grad}$ ,shrink to 1 and mult $15\times{lr}$
    - `grads = [tf.clipe_by_norm(g, 15) for g in grads]`
- Step 2:Gradient Vanishing
    - *LSTM* \ *GRU*  

##### LSTM
Compare with RNN(short term memory), which can only remenber nearly sentence.*LSTM* is long short term memory.

LSTM use three gates(sigmoid) to contral the signal. 
- Forget gate
    - $f_t = \sigma(W_f\cdot[h_{t-1}, x_t]+b_f)$
    - <img src="./static/forget_gate.png" style="zoom:50%">
- Input gate
    - $$
      \begin{aligned}
        i_t &= \sigma(W_i\cdot[h{t-1}, x_t] + b_i) \\
        \widetilde{C_t} &= tanh(W_C\cdot[h_{t-1}, x_t] + b_C)
      \end{aligned}
      $$
    - <img src="./static/input_gate.png" style="zoom:50%">
- Cell state
    - $C_t = f_f * C_{t-1} + i_t * \widetilde{C_t}$
    - <img src="./static/cell_state.png" style="zoom:50%">
- Output gate
    - $$
        \begin{aligned}
        O_t &= \sigma(W_o[h_{t-1}, x_t] + b_o) \\
        h_t &= O_t * tanh(C_t)
        \end{aligned}$$
    - <img src="./static/output_gate.png" style="zoom:50%">

##### GRU


## Auto-Encoder
Why we need:
- Dimension reduction
- Visualization
- Take advantages of *unsupervised* date
    - Unsupervise
    - *Reconstruct* itself

### Denoising AutoEncoder
Add some noise and can still reconstruct well. Means model can dig out information from a mass data.

### Dropout AutoEncoder
Use dropout to autoencoder. It the hard dropouted network can than the disdropout network do better.

### Adversarial AutoEncoder

### Variational AutoEncoder


# Gen
- Painter or Generator
- Critic or Discriminator

$$
\begin{aligned}
min_G\ max_D\ L(D,G) &= E_{x~p_r(x)}[\log{D(x)}] + E_{z~p_r(z)}[\log{1-D(G(z))}] \\
&= E_{x~p_r(x)}[\log{D(x)}] + E_{x~p_r(x)}[\log{1-D(x)}] \\
\end{aligned}
$$

Both of they want to maximum and than get a nash equilibrium

### Nash Equilibrium
- Q1.Where will D converge, given fixed G
- Q2.Where will G converge, after optimal D





### tensorflow运行机制

```python
# 本质 tf = tensor + 计算图
# tensor 数据
# op 操作
# graphs 数据操作
# session 会话核心
```

### 四则运算

```python
# 如果是变量的话要先init
tf.add(data1+data2)
tf.multiply(data1,data2)
tf.subtract(data1,data2)
tf.divide(data1,data2)

dataCopy = tf.assign(x1,x2)  # 把x2的值赋给x1
dataCopy.eval()  # 相当于sess.run(dataCopy)
# 等价于
tf.get_default_session().run(dataCopy)
```

### 矩阵运算

```pyhon
# 数据装载
x1 = tf.placeholder(tf.float32)
x2 = tf.placeholder(tf.float32)
dataAdd = tf.add(x1,x2)
sess.run(dataAdd,feed_dict={x1:2,x2:4})
# 1.tensor张量dataAdd  2.追加的数据 语法同上


## 矩阵~=数组 矩阵整体[] 每列都要[]包起来 每[]就是一行
x1 = tf.constant([2,2])
x2 = tf.constant([[2],
				  [2]])
x1.shape  #维度
sess.run(x1)   # 打印整体
sess.run(x1.[0])   # 打印第0行
sess.run(x1.[:,0])   # 打印第0列

# 运算
tf.matmul(x1,x2)  # 矩阵乘法
tf.multiply()  # 普通乘法 对应元素相乘
tf.add()   # ..

# 特殊矩阵的初始化
tf.zeros([2,3])  # 两行三列空间矩阵
tf.onex([2,3])   # 全一矩阵
tf.fill([2,3],15)  # 填充矩阵,全为15的2*3矩阵

tf.zeros_like(x1)  # 矩阵维度同x1的全零矩阵
x3 = tf.linspace(0.0,2.0,11)  # 生成一个矩阵，元素从0到2均匀分成11分
x4 = tf.random_uniform([2,3],-1,2)  # 生成2*3的一个矩阵，元素是-1到2的随机数
```

### Loss Function  

loss function:
$$loss = \sum_i(w\times x_i+b-y_i)^2 \tag{1}$$
loss 累加会很大，所以一般会除以元素个数n,结果还是一样的

$$w^` = w - lr \times \frac{\partial{loss}}{\partial{w}} \tag{2}$$
$$b^` = b - lr \times \frac{\partial{loss}}{\partial{b}}$$
这样就会得到新的w b,再返回第(1)步，如此循环就能得到最回事的w b

对loss的求导其实有规律可循:
$$\frac{\partial{loss}}{\partial{w}} = \frac{2}{n}\sum(wx + b - y)x$$
$$\frac{\partial{loss}}{\partial{b}} = \frac{2}{n}\sum(wx + b - y)$$

### Discrete Prediction
离散值预测  

Classification (分类)为例  
显然的离散的问题，那我们要怎么解决离散的问题呢？  
激活函数 activation
常见的有ReLU和sigmoid   
目的是为了把线性的值离散化，然后才能套用上面的公式  

但是就算用一个函数把线性模型离散化了，但还是太简单  
所以引入隐藏层概念  
input -> h1 -> h2 -> out  
经过多层隐藏层问题就更加离散了  
$$h1 = relu(x@w_1 + b_1)$$  
$$h2 = relu(h1@w_2 + b_2)$$  
$$out = relu(h2@w_3 + b_3)$$  
@表示矩阵乘法, 每道工序都有自己的参数   

那参数w和b怎么确定呢？  
若我们想要识别0~9,那我们是不是应该希望最后输出是有10类(一个[1,10]的矩阵,每个元素可以代表一个数字)  
那么根据矩阵运算的规则(nm\*mt = nt),所以我们只要控制每层运算符合矩阵乘法规则且最后输出是我们想要的规模就好  
最后再用out来计算loss(这里是欧氏距离(n维空间两点的距离)的loss)  
然后就可以反复更新w\` b\`了


---  


# Numpy

tensorflow的弟弟版,因为他不能GPU计算
### 基本操作
```python
x1 = np.array([第一行],[第二行]...)
x1.shape   # 打印规模
np.zeros([2,3])
np.ones([2,3])   # 零矩阵和单位矩阵的初始化（2行3列）

# 改查
x1[1,2]=5  # 第二行第一列改成5


# 基本运算
x1*x2   # 加减乘除都是对应元素加减乘除

# 矩阵运算

```

---  

# Matplotlib
`import matplotlib as plt`

### 基本操作

```python
x = np.array([1,2,3,4,5,6,7,8])
y = np.array([1,2,3,4,5,6,7,8])

# 折线图
plt.plot(x,y,"r")  # 1.x轴 2.y轴 3.颜色
plt.plot(x,y,"g",lw=10)  # 1.x轴 2.y轴 3.颜色 4.折线的宽度

# 柱状图
plt.bar(x,y,0.9,alpha=1,color='b')  # 3.柱状图的宽 4.alpha通道,即透明度
plt.show()

```













---   





