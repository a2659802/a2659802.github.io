---
layout: post
title: NMAP服务探测原理
date: 2023-03-15
categories: blog
tags: [nmap]
description: 解读nmap-service-probes文件
---


# NMAP服务探测原理

## 背景
项目原本有个用nmap做端口扫描的服务需要重写，需要了解一下其基本原理。看了一会项目发现里面有个看起来很重要的文件`fingers.conf`,里面记录了很多信息，应该是端口扫描时发送的数据内容。截取样例如下

```ini
# Nmap service detection probe list
Exclude T:9100-9107

Probe TCP NULL q||

totalwaitms 6000

tcpwrappedms 3000

ports 21,22,23,25,53,80,110,139,443,445,1080,1433,1434,1521,3128,3306,3389,5000,6379,7001,8080,9092,9080,9090,11211,27017

match 1c-server m|^S\xf5\xc6\x1a{| p/1C:Enterprise business management server/

match 4d-server m|^\0\0\0H\0\0\0\x02.[^\0]*\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0$|s p/4th Dimension database server/ cpe:/a:4d_sas:4d/
```

显然，想看懂这个文件需要去翻nmap的资料

## nmap 大体流程
https://nmap.org/book/vscan-technique.html

针对每个端口
1. 检查端口是否在排除列表，是的话跳过扫描
2. 如果端口是TCP，则会先建立连接，然后将该连接记录为open状态
3. 连接建立后，等待5秒看对端是否发送欢迎信息，如果有则匹配名为NULL的Probe.此时如果匹配到了softmatch, 则后续对该端口, 只会发送包含该服务的Probe
4. 如果NULL Probe没有匹配或者softmatch, TCP连接会执行后续的Probe。对于UDP Probe，也会在这一阶段开始
5. 为了防止前一个Probe发送的数据对后一个Probe的干扰，每个Probe都会新建一个TCP连接再发送数据

## nmap-service-probes文件格式

经过一番查找，该文件确实是用于配置探测服务时发送的数据内容，以及如何根据返回值来识别服务。

### Probe

格式 `Probe <protocol> <probename> <probestring>`

Probe 指令告诉 Nmap 发送什么字符串来识别各种服务。后面讨论的所有指令都对最近的Probe语句进行操作

<protocol>
这必须是TCP或UDP。

<probename>
探测器的英文名称。

<probestring>
告诉Nmap发送的数据，格式为：q|.....|。内容类似于C和Perl，支持转义字符：\0,\a,\n,\r。如果分隔符是你在内容中需要的，也可以换其他的分隔符。

### match 

格式 `match <service> <pattern> [<versioninfo>]`

```ini
# Example
match ftp m/^220.*Welcome to .*Pure-?FTPd (\d\S+\s*)/ p/Pure-FTPd/ v/$1/ cpe:/a:pureftpd:pure-ftpd:$1/
match ssh m/^SSH-([\d.]+)-OpenSSH[_-]([\w.]+)\r?\n/i p/OpenSSH/ v/$2/ i/protocol $1/ cpe:/a:openbsd:openssh:$2/
```

match针对最近的一个Probe指令的响应内容，使用正则匹配，判断该端口是什么服务

<service>
匹配成功的服务名称；

<pattern>
格式为：m/[regex]/[opts]；regex格式采用Perl语言格式；目前opts支持“i”，代表的含义是匹配不区分大小写；“s”：代表在‘.’字符后面有新行。

<versioninfo>
包含很多可选的字段，每个字段都由一个标识符开始，然后是分隔符包含的字段值。下面介绍7个字段：
```
　　　　p/vendorproductname/  供应商或者服务明
　　　　v/version/            应用的版本信息
　　　　i/info/　　　　　　　　 其他进一步的信息
　　　　h/hostname/　　　　　　主机名
　　　　o/operatingsystem/　　服务在什么操作系统之上
　　　　d/devicetype/　　　　　服务运行的设备类型
　　　　cpe:/cpename/[a]      nmap通用的指纹格式
```
这里可以用$1,$2等方式将pattern的group作为替换内容

### softmatch 

格式 `softmatch <service> <pattern>`

继续扫描并将Probe的扫描限定在匹配到的服务之中


### ports/sslports

在每个探针的下面,告诉Nmap探针所要发送数据的端口，仅使用一次

### Exclude 

放在所有Probe之前，指定要跳过的端口。默认配置里会排除9100-9107的端口，因为这些是打印机的常见端口。往这些端口发送数据会使打印机打印发送的数据