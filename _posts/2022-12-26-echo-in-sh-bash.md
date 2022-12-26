---
layout: post
title: echo 命令在sh和bash的不同表现
date: 2022-12-26
categories: blog
tags: [linux]
description: echo在sh和bash两种终端是有区别的
---

## 背景
工作需要实现自动化改密的功能, 一开始用ansible的user模块，发现其底层实现是用usermod命令，该命令需要root权限, 不满足需求。
后来在passwd命令的处理上找到了用echo命令来解决交互式输入的问题，但是在机器上验证一直不通过，经过排查发现是因为echo命令在sh终端上和bash终端上的不同。

## 正文
网上给出的命令都是使用`-e`参数，例如
`echo -e "$old\n$new\n$new\n" | passwd $user`

经查阅, -e参数的效果是让转义符生效
在bash终端中，不加-e参数会原封不动的打印\n，加了后就老老实实换行了

但是在sh中，不加-e本身就能识别转义字符，加了-e反而会把-e作为字符的一部分打印出来

所以在sh终端环境下需要这样写`echo "$old\n$new\n$new\n" | passwd $user `
