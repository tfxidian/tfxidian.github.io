---
title: git new repo configuration
date: 2021-08-14 16:15:50
tags: git
layout: post
---


## the key you are authenticating with has been marked as read only

出现的场景是当我新建一个repo仓库的时候，先说git clone下来github上的repo，然后要git push时出现`the key you are authenticating with has been marked as read only`这样的错误，查找了各种各样的解决方法，似乎都不太好使。

最后，解决方法如下:
- 删除~/.ssh下的文件，把原有公私钥都删除
- 重新执行命令`ssh-keygen -t rsa -b 4096 -C "[email]"`
- 把公钥写入到github配置文件中

然后，再git push 就可以了。