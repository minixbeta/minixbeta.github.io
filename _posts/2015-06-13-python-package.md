---
layout: post
title: THGP- Python 中的分发
category: 技术
comments: true
---

本文介绍了与 Python 中分发相关的要点。

## 什么是分发
当你写好了一个 Python 包，想让别人也可以方便地安装和使用你的包时，你就需要分发

## 如何分发
### setup.py
setuptools 包是分发库的主要选择，分发时，你需要使用 setuptools 包写一个 setup.py 文件，例如：

```python
from setuptools import setup, find_packages
setup(
    name = "foo",
    version = "0.1",
    packages = find_packages(),
)
```

setup.py 可以用来创建分发时的源代码包，以及 wheel 格式的包。

Wheel 格式是 Pythoon 分发包的标准，在 PEP 427 中定义。

### 打包

打包时，你可以使用下面的命令将 Python 库打包成 tarball 格式：

```
python setup.py sdist
```

也可以打包成 wheel 格式:

```
python setup.py bdist_wheel
```

### 上传至 PyPI
可以将自己的包上传至 PyPI 服务器，这样，用户安装时，只需要简单地使用下面的命令即可完成安装：

```
pip install foo
```

在正式上传前，可以先使用 PyPI 的测试服务器，测试整个流程，测试完成后，再正式启用。

整个流程是：

在 `~/.pypirc` 中添加 testpypi 节：

```
...

[testpypi]
username = <your username>
passowrd = <your password>
repository = https://testpyp.python.org/pypi
```

然后注册自己的项目：

```
python setup.py register -r testpypi
```

上传源代码：

```
python setup.py sdit upload -r testpypi
```

上传 wheel 包：

```
python setup.py bdist_wheel upload -r testpypi
```

在测试服务器上安装

安装时，使用 pip 安装

```
pip install -i https://testpyp.python.org/pypi foo
```

## 正式分发
正式分发时，在 `~/.ppirc` 中添加 `pypi` 节：

```
[pypi]
username = <your username>
password = <your passowrd>
```

然后按 testpypi 的流程分发。
