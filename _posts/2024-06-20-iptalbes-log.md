---
layout: post
title: iptables日志排查
date: 2024-06-20
categories: blog
tags: [iptables]
description: 如何排查iptables在哪个环节将包丢弃
---

## iptables日志排查

在排查一些网络底层问题时，可能需要知道iptables在哪个环节将包丢了。
这里假设读者对4表5链有简单的了解

路由转发的路径通常为：prerouting->forward->postrouting
本机程序请求外部服务通常为：output->postrouting
外部请求本机服务通常为：prerouting->input

表的处理优先级：raw>mangle>nat>filter

| 表 | 生效链 |
| --- | --- |
| filter | input,forward,output |
| nat | prerouting,output,postrouting |
| mangle | prerouting,intput,forward,output,postrouting |
| raw | output,prerouting |

由此可知，mangle在所有链上都可以添加规则，我们在上面添加一条日志规则，并结合dmesg来查看日志

```bash
root@nta-dev1:~# iptables -t mangle -A PREROUTING -p tcp --dport 8080 -j LOG --log-prefix "xxx8080pre:"
root@nta-dev1:~# iptables -t mangle -A INPUT -p tcp --dport 8080 -j LOG --log-prefix "xxx8080input:"
root@nta-dev1:~# iptables -t mangle -A FORWARD -p tcp --dport 8080 -j LOG --log-prefix "xxx8080for:"
root@nta-dev1:~# iptables -t mangle -A OUTPUT -p tcp --dport 8080 -j LOG --log-prefix "xxx8080out:"
root@nta-dev1:~# iptables -t mangle -A POSTROUTING -p tcp --dport 8080 -j LOG --log-prefix "xxx8080post:"
```

可配合`dmesg -C`来清空日志

```bash
root@nta-dev1:~# dmesg
[2020930.842761] xxx8080pre:IN=ens18 OUT= MAC=1e:59:37:4b:87:ed:0e:e1:09:68:07:75:08:00 SRC=172.16.0.114 DST=172.16.200.1 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=47985 DF PROTO=TCP SPT=60048 DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
[2020930.842856] xxx8080input:IN=ens18 OUT= MAC=1e:59:37:4b:87:ed:0e:e1:09:68:07:75:08:00 SRC=172.16.0.114 DST=172.16.200.1 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=47985 DF PROTO=TCP SPT=60048 DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
[2020930.842884] xxx8080out:IN= OUT=ens18 SRC=172.16.0.114 DST=172.16.200.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=47985 DF PROTO=TCP SPT=60048 DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
[2020930.842893] xxx8080post:IN= OUT=ens18 SRC=172.16.0.114 DST=172.16.200.1 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=47985 DF PROTO=TCP SPT=60048 DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
```

除了日志打印外，还可以通过-v参数来查看规则的命中次数是否发生变化.
pkts表示包数量

```bash
root@nta-dev1:~# iptables -t mangle -vnL
Chain PREROUTING (policy ACCEPT 9386 packets, 2769K bytes)
 pkts bytes target     prot opt in     out     source               destination
    5   276 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 LOG flags 0 level 4 prefix "xxx8080pre:"

Chain INPUT (policy ACCEPT 8295 packets, 2674K bytes)
 pkts bytes target     prot opt in     out     source               destination
    4   216 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 LOG flags 0 level 4 prefix "xxx8080input:"

Chain FORWARD (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    60 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 LOG flags 0 level 4 prefix "xxx8080for:"

Chain OUTPUT (policy ACCEPT 7918 packets, 4794K bytes)
 pkts bytes target     prot opt in     out     source               destination
   12   648 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 LOG flags 0 level 4 prefix "xxx8080out:"

Chain POSTROUTING (policy ACCEPT 7913 packets, 4794K bytes)
 pkts bytes target     prot opt in     out     source               destination
    8   432 LOG        tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 LOG flags 0 level 4 prefix "xxx8080post:"

```
