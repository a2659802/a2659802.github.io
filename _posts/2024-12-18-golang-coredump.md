---
layout: post
title: golang产生core dump的方式
date: 2024-12-18
categories: blog
tags: [linux, golang, dlv, gdb]
description: 本文介绍了如何在Golang中生成core dump文件的方法
---

## 使用dlv实现core dump

使用dlv实现起来非常简单

```bash
dlv attach <pid>

(dlv) dump <dump-path>
Dumping memory 236449792 / 236449792...
(dlv) quit
Would you like to kill the process? [Y/n] n
```

## 无任何工具修改进程的内存实现

这里提供一种完全没有gdb,dlv等工具的场景下如何产生core dump文件。
前置知识：
通过将GOTRACEBACK环境变量设为crash来启动程序的情况下, 如果ulimit设置正确, 并且有配置core dump文件产生路径, 
则可以通过`kill -SIGQUIT <pid>`来使一个go进程退出并输出core dump文件。

可以参考[这篇文章](https://www.cnblogs.com/apocelipes/p/17536722.html)

但是这里假设你的go程序是生产环境已经运行很久的程序, 那肯定不会用这样的环境变量来启动, ulimit的配置也不符合要求。
我们先从最简单的ulimit配置改起, 对于一个已经运行起来的进程, 可以通过`prlimit`来动态调整资源限制
假设进程pid=5564, 下面是命令参考

笔者的系统是ubuntu, 需要调整core_pattern来指定输出core dump文件的路径, 这里指定路径为/tmp
`sysctl -w kernel.core_pattern=/tmp/core.%e.%p`

```bash
# 默认core file size是0, 需要改为不限制
cat /proc/5564/limits
Limit                     Soft Limit           Hard Limit           Units
Max core file size        0                    unlimited            bytes
# 修改
prlimit --pid 5564 --core=unlimited
# 检查结果
root@sc:~/project/sc# cat /proc/500130/limits
Limit                     Soft Limit           Hard Limit           Units
Max core file size        unlimited            unlimited            bytes
```

接下来是最困难的环境变量调整。
总所周知, 环境变量在程序启动之后就无法修改了, 所以我们的目标是找到其他方式。
经检查代码发现, GOTRACEBACK环境变量最终会被加载到全局变量runtime.traceback_env中, 这是一个uint32类型的变量。
如果没有设置环境变量, traceback_env的值一定是4
```
runtime.traceback_env = 4 == SINGLE
runtime.traceback_env = 1 == CRASH
```

接下来要解决的问题有：
1. 如何找到这个变量的地址
2. 如何修改指定地址的值

### 寻找变量地址
- 首先, 需要拥有目标进程的源码
- 在开发环境将其使用几乎相同的参数重新编译(不能使用-s -w), 得到带调试符号的二进制 dev_binary
- 用gdb启动dev_binary, 使用`info address runtime.traceback_env`得到变量的地址dev_address, 假设进程pid为 dev_pid
- 使用`cat /proc/<dev_pid>/maps`得到内存映射
- 找到全局变量所在段的基址dev_bss, 一般通过找第一个内存的权限为`rw-p`的段
- 计算偏移var_offset=<dev_address> - <dev_bss>
- 使用同样的步骤, 得到生产环境进程的映射, 得到基址prod_bss
- 计算地址prod_address = <prod_bss> + <var_offset> 

### 编写代码修改目标地址的值
由于python在各大发行版中均有内置, 假如不需要引入第三方库, 那么即使是生产环境也能写代码。
手搓代码, 直接修改`runtime.traceback_env`变量的值

// TODO 代码有点问题
```python
import ctypes
import os

# 定义 ptrace 常量
PTRACE_ATTACH = 16
PTRACE_DETACH = 17
PTRACE_POKEDATA = 5

def write_memory(pid, address, value):
    libc = ctypes.CDLL("libc.so.6")
    libc.ptrace.argtypes = [ctypes.c_int, ctypes.c_int, ctypes.c_void_p, ctypes.c_void_p]

    # 附加到目标进程
    libc.ptrace(PTRACE_ATTACH, pid, None, None)
    os.waitpid(pid, 0)

    # 修改目标地址的数据
    libc.ptrace(PTRACE_POKEDATA, pid, ctypes.c_void_p(address), ctypes.c_void_p(value))

    # 分离目标进程
    libc.ptrace(PTRACE_DETACH, pid, None, None)

# 示例：修改某进程内存
target_pid = 1234  # 替换为目标进程的 PID
target_address = 0x7ffc12345678  # 替换为目标地址
data_to_write = 1  # 替换为要写入的值 (1 == CRASH)

write_memory(target_pid, target_address, data_to_write)
```

## 调试core dump文件

```bash
dlv core <debug_binary> <core_dump_file>
```

通常来说需要使用`grs`, `gr <goroutine_id>`, `bt`,`frame <frame_number>`等命令切换协程、堆栈

以及使用`sources`查看源代码, 并使用`config substitute-path <from> <to>`来映射代码路径