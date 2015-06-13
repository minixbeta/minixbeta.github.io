---
layout: post
title: THGP- Python 中的虚拟环境
category: 技术
comments: true
---

本文介绍了 Python 中虚拟环境相关的主题。


## 为什么需要使用虚拟环境
如果你在同时开发两个项目 A 和 B，A 依赖于 foo 1.0 包， B 依赖于 foo 2.0 包，那么系统无论安装哪个版本的包都不太合适。这时候，
可以使用虚拟环境，项目 A 和 B 分别有自己的虚拟环境。在项目 A 的虚拟环境中安装 foo 1.0 包，在项目 B 的虚拟环境中安装 foo 2.0 包。
这样也防止因为项目 A 和 B 而影响系统环境。

## 如何使用虚拟环境
使用 virtualenv 工具。

### 创建虚拟环境

```
$ virtualenv myvenv
```

### 激活虚拟环境

```
$ source myvenv/bin/activate
```

### 在虚拟环境中安装包

```
(myvenv) $ pip install foo
```

### 退出虚拟环境

```
(mynevn) $ deactivate
```
