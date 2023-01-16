---
title: FFMPEG
author: 66RING
date: 2022-07-14
tags: 
- usage
mathjax: true
---

# Abstract


# Preface


## 基本使用

自动分析后缀, 然后转换格式, **可以转到音频**, 如后缀`.mp3`

```sh
ffmpeg -i input.t1 ouput.t2
```

指定编码器

```sh
ffmpeg -i input.t1 -c:v libx264 -preset xxx -crf 22 ouput.t2
```

- `-c:v` TODO?? coder:video
- `libx264`是ffmpeg提供的一个纯软件的编码器
- `[-preset]`选择预设参数
	* 可以是fast, faster等, 编码速度和文件大小
- `[-crf]`(constant rate factor)
	* 用于控制图像质量, 0-51越大质量越差, 一般取19-28


## 过滤器

ffmpeg采用一种流式处理的方式(类似pipeling processing)，一个过滤器处理完传递给另一个过滤器。

使用`-vf`(video filter)指定过滤器

```sh
ffmpeg -i test.avi -c:v libx264 -vf "scale=1024:576" ouput.mp4
```

- `-vf "scale"`缩放过滤器
	* 指定长宽
	* 或一个参数`-1`来根据另一个参数自动推断
- `-vf "transpose=2"`旋转
	* 0, 1, 2, 3表示不同的方向
- `-vf "crop=x:h:x:y"`裁剪过滤器
	* 宽高和左上角坐标
	* 或**使用表达式** `"crop=iw/3:ih/3"`, iw(input width)的三分之一
- 过滤器组合, 使用逗号分隔: `-vf "scale=-1:720,transpose=2"`

[ffmpeg filter](https://ffmpeg.org/ffmpeg-filters.html)


## 剪切与合并

剪切视频片段

```sh
ffmpeg -i test.avi -c:v libx264 -ss 00:00:03 -t 00:00:05 ouput.mp4
```

- `-ss`指定起始位置
- `-to`指定终止位置
i `-t`表示持续时常
- **`-ss`要放在`-i`参数后面**
- 表示的格式
	* `hh:mm:ss`十分秒
	* `ss`直接给出秒数

合并视频

1. 将所有视频文件列举在一个文档中
	```
	file 'v1.mp4'
	file 'v2.mp4'
	file 'v3.mp4'
	```
2. `ffmpeg -f concat -i list.txt -c copy `
	- `-f`表示接下来的输入(`-i`)是一个视频列表
	- `-i`指定列表文件
	- `-c copy`不重新编码, 而是直接拷贝原始数据(视频格式 vs 封装格式)


## 音频处理

音频过滤器`-af`

```sh
ffmpeg -i test.mp4 -af "volume=1.5" output.mp4
```

- `-af "volume=1.5"`音量过滤器
- `-af "loudnorm=I=-5:LRA=1"`统一视频音量
- `-af "equalizar=f=1000:width_type=h:width=200:g=-10"`高通低通均衡器等


## 视频添加音频

混声

```
ffmpeg -i video.mp4 -stream_loop -1 -i audio.wav -filter_complex [0:a][1:a]amix -t 60 -y out.mp4
```

- `-stream_loop -1 -i audio.wav`, 先指定对stream的处理, 再指定的-i, 即`-i`对应的是`-i`前面的操作
- `-stream_loop -1` 参数`-1`代表循环输入源
- `[0:a][1:a]amix`将0和1号的音频流进行混合
- `-t 60` 裁剪60秒

音频替换

```
ffmpeg -an -i video.mp4 -stream_loop -1 -i audio.wav -t 60 -y out2.mp4
```

- `-an -i video.mp4`代表消除视频中的音频


## 轨道处理

```
ffmpeg -i test.mp4 -an output.mp4
```

- `-an`删除音频轨
- `-vn`删除视频轨
- `-sn`删除字幕
- `-dn`删除数据流


## 小技巧

### 创建视频缩略图

```
ffmpeg -i test.mp4 -vf "fps=1/10,scale=-2:720" name-%03d.jpg
```

- `fps=1/10`指定输出帧率, 十秒一帧
- `scale`指定输出图像的大小
- `name-%d`指定输出文件名(`printf`)


### 添加水印

```
ffmpeg -i test.mp4 -i cat.jpg -filter_complex "overlay=100:100" ouput.mp4
```

- `-i`视频输入和水印输入
- `-filter_complex "overlay=100:100"`指定叠加位置


### 屏幕录像

可以执行多个输入源，**一个输入源前的参数只会对该输入源产生作用**

```sh
ffmpeg \
	# 录制桌面
	-f x11grab -s $w'x'$h  -i "$DISPLAY+$x,$y"  \
	# 录制音频
	-f alsa -i default \
	-r $fps -c:v h264 -crf 18 -preset ultrafast \
	-c:a aac \
	$outfile
```

- `-f x11grab -s $w'x'$h  -i "$DISPLAY+$x,$y"`录制x11桌面
	* 从`$x,$y`坐标点开始录制`$w'x'$h`的值存
* `-f alsa`当前的音频系统格式


### B站视频压制

```
# 1080p
ffmpeg -i input.mp4 -preset ultrafast \
	-c:v libx264 -b:v 6000k \
#	-s 1920x1080 \
	-c:a aac -b:a 320k -crf 18 ouput.mp4
```


### 制作封面

