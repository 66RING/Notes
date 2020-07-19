---
title: openCV学习笔记
date: 2020-7-19
tags: python, opencv
---

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


## openCV的数学定义
