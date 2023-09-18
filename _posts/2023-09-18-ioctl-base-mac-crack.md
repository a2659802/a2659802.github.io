---
layout: post
title: linux下如何欺骗程序获取到虚假的MAC地址
date: 2023-09-18
categories: blog
tags: [crack,linux]
description: 通过伪造MAC地址，可以绕过部分程序的验证
---

# linux下如何欺骗程序获取到虚假的MAC地址

有的程序会绑定设备，其绑定原理有很多，这里假设这个程序是通过绑定MAC地址来绑定设备的，来研究一下怎么绕过。

## 验证程序
首先要有一个demo程序，用来获取并打印某个网卡的mac地址，代码如下：
```c
// main.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <net/if.h>

int main() {
    struct ifreq ifr;
    int sockfd;

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == -1) {
        perror("socket");
        return 1;
    }

    strcpy(ifr.ifr_name, "ens18"); // 替换为你要查询的网络接口名称

    if (ioctl(sockfd, SIOCGIFHWADDR, &ifr) == -1) {
        perror("ioctl");
        close(sockfd);
        return 1;
    }

    close(sockfd);

    unsigned char *mac = (unsigned char *)ifr.ifr_hwaddr.sa_data;

    printf("MAC Address: %02x:%02x:%02x:%02x:%02x:%02x\n",
           mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    return 0;
}
```

编译运行后，得到真正的MAC

## 伪装程序
编写一个动态链接库的代码，用来做注入

```c
// fake-hwaddr.c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <net/if.h>
#include <net/if_arp.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SIOCGIFHWADDR 0x8927

static char no_fake_hwaddr = 1;
static unsigned char hwaddr[6];
static int (*glibc_ioctl)(int, unsigned long, void *);

static void __attribute__((constructor)) init(void) {
  glibc_ioctl = dlsym(RTLD_NEXT, "ioctl");
  char *fake_hwaddr = getenv("FAKE_HWADDR");
  if (fake_hwaddr) {
    if (sscanf(fake_hwaddr, "%2hhx:%2hhx:%2hhx:%2hhx:%2hhx:%2hhx", hwaddr,
               hwaddr + 1, hwaddr + 2, hwaddr + 3, hwaddr + 4,
               hwaddr + 5) == 6 ||
        sscanf(fake_hwaddr, "%2hhx-%2hhx-%2hhx-%2hhx-%2hhx-%2hhx", hwaddr,
               hwaddr + 1, hwaddr + 2, hwaddr + 3, hwaddr + 4, hwaddr + 5) == 6)
      no_fake_hwaddr = 0;
    else
      printf("FAKE_HWADDR format error!\n");
  }
}

int ioctl(int fd, unsigned long request, void *arg) {
  if (request != SIOCGIFHWADDR || no_fake_hwaddr)
    return glibc_ioctl(fd, request, arg);
  int ret = glibc_ioctl(fd, SIOCGIFHWADDR, arg);
  if (ret != -1 && ((struct ifreq *)arg)->ifr_hwaddr.sa_family == ARPHRD_ETHER)
    memcpy(((struct ifreq *)arg)->ifr_hwaddr.sa_data, hwaddr, 6);
  return ret;
}
```

## 验证脚本
```bash
#!/bin/bash

export FAKE_HWADDR="00-A7-AC-78-B3-7C"
export LD_PRELOAD=$(pwd)/fake-hwaddr.so
$(pwd)/get-mac
```

## 编译脚本
```makefile
.PHONY: clean all

all: get-mac fake-hwaddr.so

get-mac: main.c
        ${CC} -o get-mac main.c

fake-hwaddr.so: fake-hwaddr.c
        ${CC} --shared -o fake-hwaddr.so fake-hwaddr.c -ldl -fPIC

clean:
        -rm fake-hwaddr.so
```

## 验证过程
1. 执行make编译
2. 执行./get-mac, 得到真正的MAC
3. 执行./hack.sh, 得到伪装的MAC


原理解释：
通过LD_PRELOAD劫持ioctl系统调用，返回了伪装后的结果。所以只能针对ioctl来获取mac的情况，如果走的是netlink接口，那么要劫持socket接口和recvfrom接口。