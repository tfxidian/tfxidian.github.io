---
title: Linux kernel module learning tutorial(2)
date: 2021-08-11 23:55:50
tags: Linux, kernel module
layout: post
---

## 预备知识

### 模块是怎么开始和结束的
程序通常由main函数开始，历经一系列指令终止。内核模块的各种方式不太一样，一个模块通常由init_module或者你在module_init中制定的函数开始，这是内核模块的入口。它告诉内核这个模块提供了什么功能，并设置内核在需要时运行模块的函数

