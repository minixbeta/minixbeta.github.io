---
layout: post
title: THGP- Python 中的模块和库
category: 技术
comments: true
---
本文介绍了 《Python 高手之路中》与 Python 的模块和库相关的知识

## 模块

1. sys.path 告诉 Python 从哪里导入标准库
2. 导入机制是可以自定义的，参见 [PEP 302](https://www.python.org/dev/peps/pep-0302/)

## 库

1. 标准库都有哪些内容需要完整地阅读一遍
2. 如果项目依赖外部库，确保外部库与项目的耦合度低，最好写一个包装层，使得项目的其它地方看不到外部库，只能看到包装层，
这样外部库变化时，只需要修改包装层。
3. 选择外部库的原则：
  * Python 3 兼容
  * 开发活跃
  * 维护活跃
  * 是否与流行的 linux 发行版一起打包
  * API 兼容保证
