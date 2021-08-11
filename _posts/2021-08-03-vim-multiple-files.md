---
title: vim multiple files
date: 2021-08-11 23:55:50
tags: vim
layout: post
---

在终端里输入 
```
vim file1 file2 ... filen
```
便可以打开所有想要打开的文件
同时显示多个文件：
`
`:split
:vsplit filen
`
切换命令Ctrl+w+w