---
title: Python setup.py和开发流程
author: 66RING
date: 2024-06-05
tags: 
- python
mathjax: true
---

# Python setup.py和开发流程

cheat sheet

```python
# setup.py
from setuptools import setup, find_packages
setup(
    name="pip_pkgs_name", # pip管理的包名, 如pip show pip_pkgs_name
    version="0.0.1",
    packages=["pkg1", "pkg2"], # 安装后使用的包名, 如import pkg1
)
```

临时安装, 可以打包成一个pack访问但不会安装到系统

```sh
python setup.py install
# 或
python setup.py develop
```

安装到系统, 使用`-e`参数保证可编辑，即对源文件修改后直接作用在安装过的pkg中。

```sh
pip install -e .
```

使用:

```python
import pkg1
```


