---
layout: post
title: Python 项目中需要注意哪些地方
categories : [技术，Python]
comments: true
---

## 项目相关

### Python 版本选择

必须支持 Python 2.7，如果想让项目在未来也可用，需要支持 Python 3.3 及以上

### 项目结构

```
foobar
  - docs # Sphinx 格式文档
    - conf.py
    - quickstart.rst
    - index.rst
  - foobar # 主要包
    - __init__.py
    - cli.py
    - tests # 测试用例，需要不要将其放在包外
      - test_cli.py
  - setup.py # 发布给用户时，用它来安装
  - READEME.rst # 项目说明 
  - requirements.txt # 包依赖说明
  - test-requirements.txt # 测试集包依赖说明
```

### 版本号

PEP 440 中规定版本号格式：

```
N[.N]+[{a|b|c|rc}N][.postN][.devN]
```

示例：

* 3.4: 最终版本
* 3.4a1: alpha 版本，表示版本不稳定或者缺少一些重要功能
* 3.4b1: beta 版本，功能已经完整，但是可能有 bug
* 3.4rc1: 候选版本，除非有重大 bug，否则可能成为最终发行版本
* 3.4.post2: 后续版本
* 3.4.dev3: 开发版本

### 编码风格

Python 社区提出了 PEP8 标准，对编码风格进行规范，使用自动化工具 pep8 进行检查
