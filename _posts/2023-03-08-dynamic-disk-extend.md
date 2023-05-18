---
layout: post
title: linux 虚拟机开机自动扩容实现
date: 2023-03-08
categories: blog
tags: [linux]
description: 实现linux开机扩容磁盘脚本
---

# linux 虚拟机开机自动扩容实现

## 实现原理
使用vmware等虚拟化磁盘时, 可以对磁盘文件进行扩容或者缩容. 扩容的原理是在磁盘后面追加一段未分配的空间
缩容则刚好相反，所以缩容可能会导致丢失数据，而扩容不会。

### 手动扩容实现
1. 使用fdisk删除原有分区, 然后重建分区(注意需要保证起始扇区不能变, 否则数据丢失). 然后重启
2. 修改分区表后，还需要调整文件系统. 
3. 使用resize2fs/xfs_growfs/fsadm等工具自动调整(根据文件系统类型而定(ext4,xfs等))

### 自动扩容实现
手动扩容存在两个问题: 
1. fdisk是交互式命令行, 不利于自动化
2. 需要二次重启

经过搜索, 发现有一种更好的方案：使用growpart工具
这个工具支持热扩容，不需要重启就能让硬盘分区表的更改生效
首先系统安装growpart

`yum install cloud-utils-growpart -y`

**参考步骤**

```bash
# 查看设备名
zabbix:/opt #fdisk -l

Disk /dev/sda: 26.8 GB, 26843545600 bytes, 52428800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000e953e

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     4196351     2097152   83  Linux
/dev/sda2         4196352    41943039    18873344   83  Linux

# 使用growpart扩容
localhost:~ #growpart /dev/sda 2
CHANGED: partition=2 start=4196352 old: size=37746688 end=41943040 new: size=48232415 end=52428767

# 查看sda2挂载点
zabbix:/opt #df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  3.9G     0  3.9G   0% /dev
tmpfs          tmpfs     3.9G  8.0K  3.9G   1% /dev/shm
tmpfs          tmpfs     3.9G   58M  3.8G   2% /run
tmpfs          tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda2      xfs        18G   15G  3.0G  84% /
/dev/sda1      xfs       2.0G  161M  1.9G   8% /boot


# 由于是xfs文件系统, 使用xfs_growfs命令调整
localhost:~ #xfs_growfs /
meta-data=/dev/sda2              isize=256    agcount=4, agsize=1179584 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=4718336, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 4718336 to 6029051
```

**一键脚本**
```bash
#!/bin/bash 

# check growpart
if [ -z $(which growpart) ];then
    echo "WARNNING: grow disk failed, not found growpart" 1>&2
    exit 0
fi

# run growpart 
result=$(growpart /dev/sda 2)
# check result 
if [[ "$r" == *"NOCHANGE"* ]];then
    echo "grow disk done: no change"
    exit 0
fi 
# grow fs

xfs_growfs /

exit 0
```

### 开机自动运行
为了能够达到最终的目的，需要让脚本开机时自动运行
需要借助init.d来实现。 将该脚本复制到任意地方，添加执行权限，然后执行./xxx install
```bash
#!/bin/bash
# chkconfig: 345 20 10
# Description: a tool for using 

function main()
{
    # check growpart
    if [ -z $(which growpart) ];then
        echo "WARNNING: grow disk failed, not found growpart" 1>&2
        exit 0
    fi

    # run growpart 
    result=$(growpart /dev/sda 2)
    # check result 
    if [[ "$r" == *"NOCHANGE"* ]];then
        echo "grow disk done: no change"
        exit 0
    fi 
    # grow fs

    xfs_growfs /

    exit 0
}

case $1 in
    start)
        main
    ;;
    stop)
        exit 0
    ;;
    restart)
        $0 stop && $0 start
    ;;
    status)
        exit 0
    ;;
    install)
        yum install cloud-utils-growpart -y
        cp $0 /etc/init.d/disk_auto_grow
        chmod +x /etc/init.d/disk_auto_grow
        chkconfig --add disk_auto_grow
        echo "install success"
    ;;
    *)
    # For invalid arguments, print the usage message.
    echo "Usage: $0 {start|stop|restart|install|status}"
    exit 2
    ;;
esac
```



## 参考资料
- [Resize-Extend a disk partition with unallocated disk space in Linux - CentOS, RHEL, Ubuntu, Debian & more](https://www.ryadel.com/en/resize-extend-disk-partition-unallocated-disk-space-linux-centos-rhel-ubuntu-debian/)
- [Auto expand last partition to use all unallocated space, using parted in batch mode](https://unix.stackexchange.com/questions/373063/auto-expand-last-partition-to-use-all-unallocated-space-using-parted-in-batch-m)
- [growpart Github](https://github.com/canonical/cloud-utils/blob/main/bin/growpart)
- [resize2fs: Bad magic number Stackoverflow](https://stackoverflow.com/questions/26305376/resize2fs-bad-magic-number-in-super-block-while-trying-to-open)
- [扩展系统盘的分区和文件系统（Linux）](https://support.huaweicloud.com/usermanual-evs/evs_01_0072.html#section2)