---
title: openCV学习笔记
date: 2020-7-19
tags: python, opencv
---

在openCV中图像的xy轴是这样的

``` 
(0,0) -------> x
      |
      |
      |
    y v
```


## 读取图像和视频

`import cv2`

### 图像读取

- 读取数据
    * `img_data = cv2.imread(file)`
- 显示
    * `cv2.imshow(window_name, img_data)`
    * 持久化显示：`cv2.waitKey(x_ms)`。0表示时间无限


### 视频读取

- 读取数据
    * `cap = cv2.VideoCapture(file)`
- 显示
    ``` python
    while True:
        success, img = cap.read()
        cv2.imshow("video_window", img)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break  # q退出
    ```

    
### 摄像头获取

- 读取数据
    * `cap = cv2.VideoCapture(id_of_cam)`
    * 0表示使用默认摄像头
- 设置
    * `cap.set(id, value)`
        + id=3，表示宽度
        + id=4，表示高度
        + id=10, 表示亮度
- 显示
    ``` python
    while True:
        success, img = cap.read()
        cv2.imshow("video_window", img)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break  # q退出
    ```


## 基本功能

### 转换灰度

- `imgGray = cv2.cvtColor(img_data, Color_you_want)`
    * Color使用`cv2.COLOR...`
    * 注意在openCV中图像通道是BGR，而不是RGB


### 模糊

这里以GaussianBlur为例，当然还有很多种blur方案

- `imgBlur = cv2.GaussianBlur(img_data, (kernel_size, ), sigmaX)`


### 边缘检测器

- `imgCanny = cv2.Canny(img_data, threshold1, threshold2)`
    * threshold阀值就是可能需要调整的参数，要减少边数就提高阀值
- 增加边缘的厚度：膨胀(dilate)
    * `imgDialation = cv2.dilate(imgCanny, kernel, iterations=times)`
    * kernel是一个矩阵的大小和值的阵列，可以使用numpy建立`kernel = np.ones((x, y), 数据类型)`，ones表示所有的值都是1
    * iterations=times表示迭代的次数(我们想要多厚)
- 减小边缘的厚度：侵蚀(erode与膨胀相反)
    * `imgEroded = cv2.erode(imgDialation, kernel, iterations=times)`


### 大小

- 获得大小和通道数据
    * `img.shape`，返回(height, width, BGR)
- 调整大小
    * `resized = cv2.resize(img, (width, height))`
- 裁剪
    * `img[h_begin:h_end, w_begin:w_end]`，和cv2的函数不同，这里先高度再宽度，使用数据切片来裁剪图片


### 绘制图形

图像的本质就是一个矩阵，所以我们可以使用numpy来生成矩阵，然后show出来。如`img = np.zeros((x, y))`，0表示黑色，所有将得到一个纯黑的图片。

当然用numpy生成矩阵的时候也可以指定通道数，所谓通道就是n个(x, y)的矩阵。需要注意的是opencv中通道是BGR，在python中矩阵的表示形式也需要了解

``` python
img = np.zeros((x, y, 3), np.uint8)
img[:] = 255, 0, 0  # 生成纯蓝的图片
```

- 画线
    * `cv2.line(img, (x1, y1), (x2, , y2), (B, G, R), thick)`，需要起始点和终点(x1, y1)和(x2, y2)
- 画矩形
    * `cv2.rectangle(img, (x1, y1), (x2, , y2), (B, G, R), thick)`，需要起始点和终点(x1, y1)和(x2, y2)
    * 如果font\_size部分写入`cv2.FILLED`则会填充整个矩形，同理还有很多`cv2.XXX`可以使用
- 画圆
    * `cv2.circle(img, (x1, y1), r, (B, G, R), thick)`，需要起始点和半径(x1, y1)和r
- 文本
    * `cv2.putText(img, "text", (x1, y1), font, scale, (B, G, R), thick)`
    * `font`可以使用cv2内置的`cv2.XXX`


### 变换

#### 透视变换

``` python
pts1 = np.float32([x1, y1], [x2, y2], [x3, y3], [x4, y3])
pts2 = np.float32([x1, y1], [x2, y2], [x3, y3], [x4, y3])
matrix = cv2.getPerspectiveTransform(pts1, pts2)  # 从pts1到pts2的透视变换矩阵，相当于ps中扯4个点
imgOutput = cv.warpPerspective(img, matrix, (width, heigh))
```


### 图像结合

如果有很多图像，使用图像结合就不用一张一张的的运行了，在一个窗口中完成

- 简单的合并
    * 水平合并
        + `img = np.hstack((img1, img2, ..., imgN))`
    * 垂直合并
        + `img = np.vstack((img1, img2, ..., imgN))`
    * 但单纯使用这两种简单合并存在一些问题
        + 1. 无法调节大小
        + 2. 通道数必须一样
        + 因此需要自己写些函数来有机结合他们


### 彩色图像














