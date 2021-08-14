---
title: Seccomp tutorial
date: 2021-08-14 16:15:50
tags: Seccomp
layout: post
---


## Seccomp
Seccomp是一种安全机制，通过禁止在执行过程中进行的系统调用来帮助程序员将自己的程序沙箱化.
我们将看到3种使用seccomp的方法:
- 严格模式:原始模式，不是很灵活
- 过滤模式:一个更好的版本，但仍然不是很方便
- libseccomp:一个高级API，以非常简单的方式使用seccomp

## 严格模式
Seccomp严格模式是第一个被添加到linux内核的模式。它只允许4个系统调用:read()， write()， exit()和sigreturn()。所有其他系统调用都将导致SIGKILL。
先看在使用seccomp之前的情况，例如有一个程序：

```

#include<stdlib.h>
#include<stdio.h>
#include<unistd.h>

int main()
{
	printf("this is the beginning.\n");
	fork();
	printf("see you again.\n");
	return 0;
}

```

使用gcc test.c -o test生成test，然后运行：
```
this is the beginning.
see you again.
see you again.
```
由结果可见，fork可以正常使用。

然后试一下严格模式下的seccomp,使用方式很简单：
- 在头文件中加入sys/prctl.h和linux/seccomp.h
- 在文件中加入`prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);`

```
#include<stdlib.h>
#include<stdio.h>
#include<unistd.h>
#include<sys/prctl.h>
#include<linux/seccomp.h>

int main()
{
	printf("this is the beginning.\n");
	prctl(PR_SET_SECCOMP,SECCOMP_MODE_STRICT);
	fork();
	printf("see you again.\n");
	return 0;
}

```

同样编译运行，执行发现下面输出：
```
this is the beginning.
Killed
```

