---
title: virtualenvwarpper安装与配置
categories:
  - tool
tags: virtualenv
mathjax: false
date: 2021-06-03 11:00:50
---



# Virtualenvwrapper
`virtualenvwarpper`一般用作`python`管理不同的env。把`virtualenv`拓展之后，用起来更加简单方便


1. 可以将所有虚拟环境环境放在一个目录下，方便管理
2. 切换虚拟环境较为方便
3. 管理虚拟环境比较方便


# Installation

```bash
pip3 install virtualenvwrapper

# 创建目录用来存放虚拟环境
mkdir ~/.virtualenvs

# 在.bashrc中添加
## 指定PYTHON 版本
VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
export WORKON_HOME=~/.virtualenvs
source /usr/local/bin/virtualenvwrapper.sh

# 运行
source ~/.bashrc
```


- workon [虚拟环境名称]:切换虚拟环境
- lsvirtualenv:列出已有的虚拟环境列表
- mkvirtualenv :新建虚拟环境
- rmvirtualenv :删除虚拟环境
- deactivate: 离开已进入的虚拟环境