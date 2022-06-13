---
title: Imagic Magic Cheat sheet
author: 66RING
date: 2021-05-27
tags: 
- cheat sheet
mathjax: true
---


- 查看图片信息
	* `identify`
- 格式转换：自动识别扩展名
	* `convert in.<any> out.<any>`
- 压缩`-quality`
	- `convert -quality 85 <src> <dst>`
- 修改大小`-resize <width>x<hight>[!]`
	* `!`表示不改变原比例
	* 宽或高省略一个时，另一个根据比例自动计算
- 合并：e.g.
	* `-size <w>x<h>`新建画布
	* `-geometry +<h>+<w> -composite <img>`位置h,w处放img
```sh
convert -size 512x512 -strip -colors 8 -depth 8 xc:none u0.png -geometry +0+0 -composite u1.png -geometry +256+0 -composite d0.png -geometry +0+256 -composite d1.png -geometry +256+256 -composite dest4.png
```

https://www.jianshu.com/p/0c85c8c90239

