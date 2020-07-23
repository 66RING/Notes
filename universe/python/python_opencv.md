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


### 检测颜色

- 色彩捕获
- 1. 将图像转换成HSV`imgHSV = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)`
- 2. 编写TrackBars以辅助我们找出最佳值
    ``` python
    # 新建一个窗口
    cv2.nameWindow("TrackBars")
    cv2.resizeWindow("TrackBars", 640, 240)  # 通过winName找到想要的win

    # 创建跟踪栏
    # cv2.createTrackbar("barName", "winName", min, max, func)
    # func接受一个值，当Trackbar的值改变值执行对应操作
    # 这样的trackbar有6个，包含所有的色相最大最小饱和度等
    cv2.createTrackbar("Hue min", "TrackBars", 0, 179, func)
    cv2.createTrackbar("Hue max", "TrackBars", 179, 179, func)
    cv2.createTrackbar("Sat min", "TrackBars", 0, 255, func)
    cv2.createTrackbar("Sat max", "TrackBars", 255, 255, func)
    cv2.createTrackbar("val min", "TrackBars", 0, 255, func)
    cv2.createTrackbar("val max", "TrackBars", 255, 255, func)
    ```
    * 跟踪trackbar的值`val = cv2.getTrackbarPos("barName", "winName")`
    * 无限循环以实时更新
- 3. 创建mask遮罩`mask = cv2.inRange(img, lower, upper)`
    * `lower = np.array([min1, min2, ..., minN])`
    * `upper = np.array([max1, max2, ..., maxN])`
- 4. 把遮罩显示出来调整trackbar，以找到合适的值
- 5. 使用遮罩和原始图片创建新图片即可得到抽取颜色后的图片
    * `res = cv2.bitwise_and(img, img, mask=mask)`，and:按位与操作
- 堆叠图像，而不是一个一个显示所有图像


### 检查图形

- 1. 转化成灰度图，以便找到拐角
    * `imgGray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)`
- 2. 添加一些模糊
    * `imgBlur = cv2.GaussianBlur(imgGray, (kernel,), sigma)`
- 3. 描边
    * `imgCanny = cv2.Canny(imgBlur, threshold, threshold)`
- 4. 检测轮廓

    ``` python
    def getContours(img):
        contours, hierarchy = cv2.findContours(img, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_NONE)
        for cnt in contours:
            area = cv2.contoursArea(cnt)
            if area>500:  # 通过面积来消除噪声
                cv2.drawContours(drawToImg, cnt, -1, (B, G, R), thick) # -1 画所有轮廓
                # 计算轮廓长度
                peri = cv2.arcLength(cnt, True)  # true表示封闭
                # 计算拐点位置
                approx = cv2.approxPolyDP(cnt, 0.02*peri(分辨率), True)
                # 计算角
                objCor = len(approx)

                # 画边框，实现有个方框包围的效果
                x, y, w, h = cv2.boundingRect(approx)
                cv2.rectangle(drawToImg, (x, y), (x+w, y+h), (B, G, R))
                
    ```


### 检查面部

opencv使用训练得到的级联(Cascade)文件来处理图像。opencv提供了一些默认的Cascade，可以处理人脸、车牌、眼睛等等。

- 引入Cascade，如
    * `faceCascade = cv2.CascadeClassifier("path/to/Cascade/file")`
- 把图像换成灰度图
- 使用引入的Cascade处理图像
    * `faces = faceCascade.detectMultiScale(img, scale, minimum_neighbor)`
- 画框框
    ``` python
    for (x, y, w, h) in faces:
        cv2.recrangle(img, (x, y), (x+w, y+h), (B, G, R), thick)
    ```

## 使用

[源码](https://github.com/murtazahassan/Learn-OpenCV-in-3-hours)

### 检查画笔颜色并在图像中画出

- 1. [颜色检测](#检测颜色)，创建蒙版
    * 为同时应对多种色彩，可以将检查到的颜色的参数记录在一个列表中
- 2. 试着[描出颜色的轮廓](#检查图形)
    * 只是为了给出明显的反馈，用完可以删除
- 3. 使用轮廓的参数如x，y就可确定一点的位置
- 4. 然后就可画出想要的BGR颜色
    * 创建一个点的列表，每次绘制出所有点[x, y, color]


### 物体扫描

- 预处理
    * [调整大小](#摄像头获取)
    * [使用灰度图](#转换灰度)
    * [模糊](#模糊)
    * [描边](#边缘检测器)
        + 调整边框粗细等
- 找到主体
    * 预处理好的图像该如何找到主体?
    * 可通过[拐角数](#检查图形)、最大面积等判断
- 通过主体的拐角进行[变换](#变换)


### 车牌识别

- 1. [引入Cascade](#检查面部)，用于识别车牌号
- 2. 框出
- 3. 把识别出来的[数字放在图片的适当位置](#绘制图形)
- 4. 保存识别的图片
    ``` python
    cv2.imwrite("path/to/file", img)
    ```

















