---
layout: post
title: 麒麟银河V10虚拟化管理
date: 2025-02-28
categories: blog
tags: [linux, kylin, qemu, kvm, arm]
description: kylin v10 sp3 + 鲲鹏920
---

## 上传镜像文件

考虑到镜像较大, 传输很耗时, 需要在环境准备工作开始阶段一边传输一边做其他事情

```bash
[root@localhost ~]# ls /root/Kylin-Server-V10-SP3-General-Release-2303-ARM64.iso
/root/Kylin-Server-V10-SP3-General-Release-2303-ARM64.iso
```

## 安装依赖

```bash
[root@localhost ~]# dnf groups list ids
上次元数据过期检查：1:24:57 前, 执行于 2025年02月28日 星期五 08时35分12秒。
可用环境组：
   最小安装 (minimal-environment)
   基础设施服务器 (server-product-environment)
   文件及打印服务器 (file-print-server-environment)
   基本网页服务器 (web-server-environment)
   虚拟化主机 (virtualization-host-environment)
已安装的环境组：
   带 UKUI GUI 的服务器 (kylin-desktop-environment)
已安装组：
   容器管理 (container-management)
   无图形终端系统管理工具 (headless-management)
可用组：
   麒麟安全增强工具 (kysecurity-enhance)
   开发工具 (development)
   传统 UNIX 兼容性 (legacy-unix)
   科学记数法支持 (scientific)
   安全性工具 (security-tools)
   系统工具 (system-tools)
   智能卡支持 (smart-card)
   Man 手册 (man-help)
[root@localhost ~]# dnf group install virtualization-host-environment

[root@localhost ~]# yum install virt-install
```

## 查看虚拟化支持

```bash
[root@localhost ~]# dmesg | grep kvm
[    0.617075] kvm [1]: Hisi ncsnp: enabled
[    0.617170] kvm [1]: 16-bit VMID
[    0.617171] kvm [1]: IPA Size Limit: 48bits
[    0.617194] kvm [1]: GICv4 support disabled
[    0.617195] kvm [1]: vgic-v2@9b020000
[    0.617209] kvm [1]: GIC system register CPU interface enabled
[    0.617579] kvm [1]: vgic interrupt IRQ1
[    0.617990] kvm [1]: VHE mode initialized successfully
```
注： 网上很多文章说的lscpu之类的检查指令集方式并不适用, 这里关注VHE即可

## 修改配置

修改 /etc/libvirt/qemu.conf
```conf
user = "root"
# The group for QEMU processes run by the system instance. It can be
# specified in a similar way to user.
group = "root"

```

```bash
[root@localhost ~]# vim /etc/libvirt/qemu.conf
[root@localhost ~]# systemctl restart libvirtd
[root@localhost ~]# systemctl enable libvirtd
```

## 检查是否正常

```bash
[root@localhost ~]# virsh version
根据库编译：libvirt 6.2.0
使用库：libvirt 6.2.0
使用的 API: QEMU 6.2.0
运行管理程序: QEMU 4.1.0

[root@localhost ~]# virsh list
 Id   名称   状态
-------------------

[root@localhost ~]#
```

## 创建存储池

```bash
[root@localhost ~]# virsh pool-define-as local1 --type dir --target /home/kvm/images
定义池 local1

[root@localhost ~]# virsh pool-build local1
构建池 local1

[root@localhost ~]# virsh pool-start local1
池 local1 已启动

[root@localhost ~]# virsh pool-autostart local1
池 local1 标记为自动启动

[root@localhost ~]# virsh pool-list
 名称     状态   自动开始
---------------------------
 local1   活动   是

[root@localhost ~]# virsh pool-info local1
名称：       local1
UUID:           ba4d3202-0769-45c2-8625-7b100ed81fda
状态：       运行中
持久：       是
自动启动： 是
容量：       363.02 GiB
分配：       13.74 GiB
可用：       349.28 GiB
```

## 创建网桥br1

kvm似乎有一个默认的virbr0网桥, 这里创建一个br1可以后续用于桥接（如果网络环境支持）

```
[root@localhost ~]# brctl addbr br1
[root@localhost ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp125s0f0       UP             172.16.88.125/23 fe80::4eee:8ab4:e9c4:6028/64
enp125s0f1       DOWN
enp125s0f2       DOWN
enp125s0f3       DOWN
virbr0           DOWN           192.168.122.1/24
virbr0-nic       DOWN
br1              DOWN
```

## 创建虚拟机

### 分配虚拟磁盘

```bash
[root@localhost ~]# virsh vol-create-as --pool local1 --name kylinv10.img --capacity 40G --allocation 1G --format qcow2
已创建卷 kylinv10.img

[root@localhost ~]# virsh vol-info /home/kvm/images/kylinv10.img
名称：       kylinv10.img
类型：       文件
容量：       40.00 GiB
分配：       196.00 KiB
```

### 虚拟机安装

自行调整各项参数的值

```bash
virt-install \
	--name=vm_kylinv10 \
	--vcpus=8 --ram=8192 \
	--disk path=/home/kvm/images/kylinv10.img,format=qcow2,size=40,bus=virtio \
	--cdrom /root/Kylin-Server-V10-SP3-General-Release-2303-ARM64.iso \
	--network bridge=br1,model=virtio \
    --graphics spice,listen=0.0.0.0 \
    --noautoconsole 
```

#### 开放spice端口

```bash
[root@localhost ~]# firewall-cmd --zone=public --add-port=5900/tcp --permanent
success
[root@localhost ~]# firewall-cmd --reload
success
```

#### 开机同意协议
安装完成首次开机会要求同意用户协议, 按照命令行提示勾选`I accept the license aggrement`, 然后continue
直到进到系统

#### 虚拟机网络配置

在本机(windows)安装`virt-viewer`, 使用spice://<宿主机ip>:5900连上去, 在图形化界面完成安装

修改虚拟机的网卡配置, 将以下配置填好, 然后执行`systemctl restart NetworkManager`应用配置

```ini
BOOTPROTO=static
ONBOOT=yes
IPADDR=
NETMASK=
GATEWAY=
DNS1=
```

#### 异常处理

1. 安装出问题, 想删除重来

```bash 
virsh dumpxml vm_kylinv10 | grep nvram
rm -f /var/lib/libvirt/qemu/nvram/vm_kylinv10_VARS.fd
virsh shutdown vm_kylinv10
virsh undefine vm_kylinv10
virsh vol-delete --pool local1 kylinv10.img 
```


后续可以在宿主机使用console命令操作虚拟机的终端(桥接还没弄好或者有问题等无法ssh的情况)

### 网桥配置

接下来的操作有一定的危险性, 失败的情况下会导致**宿主机网络断开**,如果没有非SSH访问宿主机的手段,
建议不要轻易设置桥接, 以免发生意外。

编辑`/etc/sysconfig/network-scripts/ifcfg-br1`
```ini
BOOTPROTO=static
NAME=br1
DEVICE=br1
TYPE=Bridge
ONBOOT=yes
IPADDR=172.16.88.125
NETMASK=255.255.254.0
GATEWAY=172.16.88.254
```

编辑`/etc/sysconfig/network-scripts/ifcfg-enp125s0f0`
删除原来的IP配置, 将ONBOOT设为yes,BOOTPROTO改为none,  添加BRIDGE配置
```ini
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp125s0f0
UUID=d7c01a3d-8aca-4c59-89ec-7c7a50059bce
DEVICE=enp125s0f0
ONBOOT=yes
#IPADDR=172.16.88.125
#PREFIX=23
#GATEWAY=172.16.88.254
#DNS1=192.168.40.1
IPV6_PRIVACY=no
BRIDGE=br1
```

```bash
#!/bin/bash
systemctl restart NetworkManager
brctl stp br1 on
brctl addif br1 enp125s0f0
ip addr add 172.16.88.125/23 dev br1
systemctl restart NetworkManager
ifup br1
```

## 虚拟机调整

### 移除spice相关配置

找到类似这样的配置, 将其删除

```xml
<graphics type='spice' port='5900' autoport='yes' listen='0.0.0.0'>
  <listen type='address' address='0.0.0.0'/>
</graphics>
```


### 虚拟机克隆

在虚拟机关机状态下

```bash
virt-clone -o vm_kylinv10 -n vm89140 -f /home/kvm/images/vm89140.img
virt-clone -o vm_kylinv10 -n vm89141 -f /home/kvm/images/vm89141.img
```

### 修改虚拟机配置

```bash
virsh setvcpus vm89140 16 --config --maximum
virsh setmaxmem vm89140 16G --config
```

注意最大cpu和最大内存必须重启虚拟机才能生效,运行时可以通过`setmem`,`setvcpus`来动态调整

```bash
virsh setmem vm89140 8G --live
virsh setvcpus vm89140 4 --live
virsh dominfo vm89140
```

### 添加第二张网卡
有时候需要加一张另一个网段的卡, 这里添加一个用来表示业务网口的网卡

#### 临时添加

`virsh attach-interface <vm_name> --type network --source default --model virtio`

也可以增加--persistent表示配置持久化

`virsh attach-interface <vm_name> --type network --source default --model virtio --persistent`

确认网卡已添加

`virsh domiflist <vm_name>`

#### 手动编辑

`virsh edit <vm_name>`

找到`interface`模仿原来的添加一个



### 修改虚拟机IP 

选择一个该网段没有被占用的IP, 例如172.16.89.142

这里不知道为啥重启`NetworkManager`没有自动修改, 只好手动用`ifdown`,`ifup`重启网卡. 改完之后在外面`ping`一下试试能通

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enp3s0
[root@kylin-kvm ~]# systemctl restart NetworkManager
[root@kylin-kvm ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp3s0           UP             172.16.88.244/23 fe80::5e9f:7456:b0b6:1bb3/64 fe80::2c0e:7f5:f54:700a/64 fe80::e20d:2cd6:7011:49ad/64
[root@kylin-kvm ~]# ifdown enp3s0
成功停用连接 "enp3s0"（D-Bus 活动路径：/org/freedesktop/NetworkManager/ActiveConnection/1）
[root@kylin-kvm ~]# ifup enp3s0
连接已成功激活（D-Bus 活动路径：/org/freedesktop/NetworkManager/ActiveConnection/2）
[root@kylin-kvm ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp3s0           UP             172.16.89.142/23 fe80::5e9f:7456:b0b6:1bb3/64 fe80::2c0e:7f5:f54:700a/64 fe80::e20d:2cd6:7011:49ad/64
[root@kylin-kvm ~]#
```


### 不关机调整磁盘大小

```bash

# 不可行的方法示例
[root@localhost images]# virsh vol-resize vm89140.img 80G --pool local1
错误：Failed to change size of volume 'vm89140.img' to 80G
Is another process using the image [/home/kvm/images/vm89140.img]?

[root@localhost images]# qemu-img resize /home/kvm/images/vm89140.img +40G
qemu-img: Could not open '/home/kvm/images/vm89140.img': Failed to get "write" lock
Is another process using the image [/home/kvm/images/vm89140.img]?


# 可行的在线扩容命令
[root@localhost images]# virsh blockresize vm89140 /home/kvm/images/vm89140.img 80G

# 进入虚拟机操作, 查看分区
[root@localhost images]# virsh console vm89140
[root@kylin-kvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                      253:0    0   80G  0 disk
├─vda1                   253:1    0  600M  0 part /boot/efi
├─vda2                   253:2    0    1G  0 part /boot
└─vda3                   253:3    0 38.4G  0 part
  ├─klas_kylin--kvm-root 252:0    0 34.4G  0 lvm  /
  └─klas_kylin--kvm-swap 252:1    0    4G  0 lvm  

# growpart拓展分区
[root@kylin-kvm ~]# yum install cloud-utils-growpart

[root@kylin-kvm ~]# growpart /dev/vda 3
CHANGED: partition=3 start=3328000 old: size=80556032 end=83884032 new: size=164444127 end=167772127
[root@kylin-kvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                      253:0    0   80G  0 disk
├─vda1                   253:1    0  600M  0 part /boot/efi
├─vda2                   253:2    0    1G  0 part /boot
└─vda3                   253:3    0 78.4G  0 part
  ├─klas_kylin--kvm-root 252:0    0 34.4G  0 lvm  /
  └─klas_kylin--kvm-swap 252:1    0    4G  0 lvm  [SWAP]

[root@kylin-kvm ~]# pvresize /dev/vda3
  Physical volume "/dev/vda3" changed
  1 physical volume(s) resized or updated / 0 physical volume(s) not resized
[root@kylin-kvm ~]# vgdisplay klas_kylin-kvm
  --- Volume group ---
  VG Name               klas_kylin-kvm
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               78.41 GiB
  PE Size               4.00 MiB
  Total PE              20073
  Alloc PE / Size       9833 / 38.41 GiB
  Free  PE / Size       10240 / 40.00 GiB
  VG UUID               J2i11Y-1vTX-INwT-tsRy-aTfO-VQZS-6DYam5

[root@kylin-kvm ~]# lvextend -l +100%FREE /dev/klas_kylin-kvm/root
  Size of logical volume klas_kylin-kvm/root changed from 34.41 GiB (8809 extents) to 74.41 GiB (19049 extents).
  Logical volume klas_kylin-kvm/root successfully resized.
# 确认分区类型, 决定使用xfs_growfs命令还是resize2fs命令
[root@kylin-kvm ~]# df -T
文件系统                         类型        1K-块    已用     可用 已用% 挂载点
/dev/mapper/klas_kylin--kvm-root xfs      36064048 7880836 28183212   22% /
/dev/vda2                        xfs       1038336  197336   841000   20% /boot
/dev/vda1                        vfat       613184    6660   606524    2% /boot/efi
[root@kylin-kvm ~]# xfs_growfs /
meta-data=/dev/mapper/klas_kylin--kvm-root isize=512    agcount=4, agsize=2255104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=9020416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=4404, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 9020416 to 19506176
# 确认扩容结果
[root@kylin-kvm ~]# df -h
文件系统                          容量  已用  可用 已用% 挂载点
/dev/mapper/klas_kylin--kvm-root   75G  7.8G   67G   11% /
/dev/vda2                        1014M  193M  822M   20% /boot
/dev/vda1                         599M  6.6M  593M    2% /boot/efi
```

## 快照

### 创建快照

含内存

`virsh snapshot-create-as --domain vm89142 --name snap1 --description "My first snapshot" --atomic --live`

仅磁盘(外部快照, virsh不支持恢复)

`virsh snapshot-create-as --domain vm89142 --name snap2 --disk-only --atomic`

### 快照回滚

`virsh snapshot-revert --domain [domain] [snapshot-name]`

## 安全策略

为了模拟客户没有外网的环境，需要限制虚拟机的联网，最简单的方式是创建虚拟机时，连接无法上网的网桥即可。不过这样不方便在办公网进行SSH连接。
这里尝试针对特定域名/ip进行拦截，例如镜像仓库的，用于测试某些组件安装时是否依赖了在线的镜像源。

比较简单的做法是影响虚拟机的DNS设置，例如修改虚拟机的hosts文件，将需要屏蔽的域名解析到127.0.0.1上

## 快速克隆脚本

```bash
#!/bin/bash
# 脚本名称：quick_clone
# 功能：快速克隆KVM虚拟机并自动配置网络
# 用法：./quick_clone <新虚拟机名称> <内存大小> [IP地址]
# 其中名字为vmXXYYY时，可以会根据XXYYY自动设置IP

# 检查参数
if [ $# -lt 2 ]; then
  echo "Usage: $0 <vm_name> <memory_size> [ip_address] [disk_resize] [cpu_max]"
  exit 1
fi

TEMPLATE_VM="vm_kylinv10"       # 模板虚拟机名称
IFACE_NAME="enp1s0"
IMAGE_DIR="/home/kvm/images"    # 镜像目录
TEMPLATE_IP="172.16.89.0"     # 模板虚拟机IP
NEW_VM="$1"
MEMORY="$2"
TARGET_IP="$3"
DISK_RESIZE="$4"
CPU_MAX="$5"

if [ -z "$TARGET_IP" ]; then
  # 从名称中提取最后两部分数字（例如 vm89142 -> 89.142）
  #IP_SUFFIX=$(echo "$NEW_VM" | grep -oP '\d{2}\d{3}$' | sed 's/\(..\)\(...\)/\1.\2/')
  # 改为个位数的vm895->89.5
  IP_SUFFIX=$(echo "$NEW_VM" | grep -oP '\d{2}\d{1}$' | sed 's/\(..\)\(.\)/\1.\2/')
  
  # 检查是否成功提取
  if [ -z "$IP_SUFFIX" ]; then
    echo "Error: Unable to extract IP suffix from VM name. Please provide IP address manually."
    exit 1
  fi

  # 拼接完整的 IP 地址
  NEW_IP="172.16.$IP_SUFFIX"
else
  NEW_IP="$TARGET_IP"
fi

# 1. 克隆虚拟机
virt-clone -o "$TEMPLATE_VM" -n "$NEW_VM" -f "$IMAGE_DIR/$NEW_VM.img"

# 3. 调整虚拟机配置
if [ -n "$CPU_MAX" ]; then
  virsh setvcpus "$NEW_VM" $CPU_MAX --config --maximum
fi
virsh setmaxmem "$NEW_VM" "$MEMORY" --config

# 2. 启动新虚拟机
virsh start "$NEW_VM"

virsh setmem "$NEW_VM" "$MEMORY" --live
if [ -n "$CPU_MAX" ]; then
  virsh setvcpus "$NEW_VM" $CPU_MAX --live
fi

# 3. 可选：调整虚拟机磁盘大小
if [ -n "$DISK_RESIZE" ]; then
  virsh blockresize "$NEW_VM" "$IMAGE_DIR/$NEW_VM.img" "$DISK_RESIZE"
fi

# 4. 等待虚拟机启动
echo "Waiting for VM to start and SSH service to be available..."
MAX_RETRIES=10  # 最大重试次数
RETRY_INTERVAL=10  # 每次重试间隔（秒）

for i in $(seq 1 $MAX_RETRIES); do
  echo "Attempt $i: Trying to connect to $TEMPLATE_IP..."
  if ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no root@$TEMPLATE_IP true; then
    echo "SSH connection successful!"
    break
  else
    echo "SSH connection failed. Retrying in $RETRY_INTERVAL seconds..."
    sleep $RETRY_INTERVAL
  fi

  if [ $i -eq $MAX_RETRIES ]; then
    echo "Error: Failed to connect to VM after $MAX_RETRIES attempts. Exiting."
    exit 1
  fi
done

# 5. 修改网络配置（通过SSH）

TEMP_SCRIPT=$(mktemp)  # 创建临时脚本文件
cat > "$TEMP_SCRIPT" <<EOF
#!/bin/bash
# 修改网络配置
sed -i "s/IPADDR=.*/IPADDR=$NEW_IP/" /etc/sysconfig/network-scripts/ifcfg-$IFACE_NAME
ifdown $IFACE_NAME && systemctl restart NetworkManager
hostnamectl set-hostname $NEW_VM
EOF

if [ -n "$DISK_RESIZE" ]; then
  cat >> "$TEMP_SCRIPT" <<EOF
df -h
growpart /dev/vda 3
pvresize /dev/vda3
vgdisplay klas_kylin-kvm
lvextend -l +100%FREE /dev/klas_kylin-kvm/root
xfs_growfs /
df -h
EOF
fi

scp -o StrictHostKeyChecking=no "$TEMP_SCRIPT" root@$TEMPLATE_IP:/tmp/configure_network.sh
ssh -o StrictHostKeyChecking=no root@$TEMPLATE_IP chmod +x /tmp/configure_network.sh
ssh -o StrictHostKeyChecking=no root@$TEMPLATE_IP /tmp/configure_network.sh &

# 8. 清理本地临时脚本
rm -f "$TEMP_SCRIPT"

# 6. 验证新IP
echo "New VM $NEW_VM configured with IP: $NEW_IP"
echo "Try to connect: ssh root@$NEW_IP"
ping -c 3 $NEW_IP
```

## 快速删除脚本
```bash
#!/bin/bash
POOL_NAME="local1"              # 存储池名称

if [ $# -lt 1 ]; then
  echo "Usage: $0 <vm_name>"
  echo "Warn: should shutdown vm manually before remove"
  exit 1
fi
VM_NAME="$1"

rm -f /var/lib/libvirt/qemu/nvram/${VM_NAME}_VARS.fd
virsh undefine ${VM_NAME}
virsh vol-delete --pool ${POOL_NAME} ${VM_NAME}.img 
```

## 网卡流量镜像

假如需要将宿主机的某个网卡入口流量镜像到某个虚拟机的网卡上，可以通过下面步骤实现

确认宿主机上的目标网卡, 这里是`vnet7`

```bash
[root@localhost ~]# virsh domiflist vm89146
 接口    类型      源        型号     MAC
---------------------------------------------------------
 vnet6   bridge    br1       virtio   52:54:00:f0:xx:xx
 vnet7   network   default   virtio   52:54:00:25:xx:xx
```

确认源网卡，可以用tcpdump之类的方式确认，这里是`enp125s0f3`

使用`tc`命令实现流量镜像

```bash
# 加载内核模块
modprobe sch_ingress
modprobe act_mirred

# 在源网卡创建ingress规则
tc qdisc add dev enp125s0f3 handle ffff: ingress
# 添加镜像规则
tc filter add dev enp125s0f3 parent ffff: protocol all u32 match u32 0 0 action mirred egress mirror dev vnet7
# 检查规则
tc filter show dev enp125s0f3 parent ffff:
```

注意宿主机网卡要工作在混杂模式，否则很多流量镜像不过去
`ip link set enp125s0f3 promisc on`

不需要之后，可以清理规则
```bash
tc qdisc del dev enp125s0f3 ingress
```

## 环境重建

### 虚拟化搭建
按照之前的步骤，把虚拟化平台搭好，网桥之类的配好，存储池起相同的名字

### 使用img文件创建虚拟机

这里直接使用--import导入保存的img文件

```bash
virt-install \
    --name=vm_kylinv10 \
    --vcpus=8 --ram=8192 \
    --disk path=/home/kvm/images/kylinv10.img,format=qcow2,bus=virtio \
    --import \
    --network bridge=br1,model=virtio \
    --noautoconsole
```

### 修正网络配置

上一步导入后会自动开机，进去后发现虚拟机内部的网卡名称发生变化，原来的配置需要调整

1. 修改`/etc/sysconfig/network-scripts/ifcfg-enp3s0`的名字为新的网卡名，例如`/etc/sysconfig/network-scripts/ifcfg-enp1s0`
2. 编辑`ifcfg-enp1s0`文件，调整`NAME`字段和`DEVICE`字段
3. 根据需要调整IP地址、掩码、网关等
4. 执行`nmcli con reload` 和 `nmcli con up enp1s0`应用配置