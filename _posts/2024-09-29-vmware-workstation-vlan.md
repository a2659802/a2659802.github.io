---
layout: post
title: vmware workstation尝试创建trunk模式的虚拟网卡
date: 2024-09-29
categories: blog
tags: [linux, vmware, hyper-v, vlan]
description: 借助hyper-v的功能来实现vmware workstation绑定trunk模式网卡
---

## vmware workstation尝试创建trunk模式的虚拟网卡

由于需要验证一些VLAN相关的功能，需要有TRUNK模式的网卡，故需要通过虚拟化来模拟对应的场景

vmware workstation本身无法支持创建这样的网卡，必须借助另一个虚拟化平台Hyper-V来实现。
由于和本文内容关系不大，这里暂时不讨论为什么不直接使用Hyper-V。

### 验证hyper-v过程

通过powershell命令行可以创建虚拟交换机和虚拟网卡，并且设置网卡的模式
虚拟交换机的创建也可以通过GUI界面完成，所以这里略过相关步骤。

假设已经创建好了一个虚拟交换机，名为vlan
```powershell
PS C:\Windows\system32> get-vmnetworkadaptervlan -ManagementOS

VMName VMNetworkAdapterName   Mode     VlanList
------ --------------------   ----     --------
       vlan                   Untagged
       Container NIC 0c260408 Untagged
       Container NIC e1db3f73 Untagged
```

通过下面两条命令，可以创建一个名为`trunk-x`并且连接到`vlan`交换机的网卡，其模式为Trunk, 默认VLAN为7
```powershell
add-vmnetworkadapter -managementos -name trunk-x -switchname vlan
set-vmnetworkadaptervlan -managementos -vmnetworkadaptername trunk-x -Trunk -allowedvlanidlist 1-4094 -nativevlanid 7
```

```powershell
get-vmnetworkadaptervlan -managementos

VMName VMNetworkAdapterName   Mode     VlanList
------ --------------------   ----     --------
       trunk-x                Trunk    7,1-4094
       vlan                   Untagged
       Container NIC 0c260408 Untagged
       Container NIC e1db3f73 Untagged
```

打开<虚拟网络编辑器>，修改vmware网络配置，创建一个Vmnet7桥接到trunk-x，然后分配给虚拟机

通过创建对应网卡，使用ping -I验证，发现vmware虚拟机之间可正常传递带VLAN TAG消息，但是消息无法传递到hyper-v的虚拟机

### 验证vmware仅主机模式过程

上面验证的结果不算理想，如果不能互通的话，应该有办法只依赖vmware也能验证相关功能，找了一圈发现可以在网络适配器里面将VLAN ID改为4095即可允许TAG消息传递。

1. 打开虚拟网络编辑器
2. 添加一个仅主机模式的网络，这里为VMNet9，关闭DHCP功能
3. 打开控制面板-网络和Internet-网络连接(其实就是更改适配器设置那里)，找到VMware Network Adapter VMnet9
4. 右键-属性-配置-高级-VLAN ID，将值改为4095（似乎其他值也行，可自行验证），然后确定保存

#### (可选)减少无关流量干扰

1. 修改宿主机网卡属性，取消勾选Internet协议版本4和6
2. 取消勾选LLDP，链路层等勾选项



### 参考

- [set-vmnetworkadaptervlan命令](https://learn.microsoft.com/zh-cn/powershell/module/hyper-v/set-vmnetworkadaptervlan?view=windowsserver2022-ps)
- [VMware Workstation网络修改vlan id值](https://www.cnblogs.com/one99/p/9612850.html)
- [vshpere文档](https://docs.vmware.com/cn/VMware-vSphere/7.0/com.vmware.vsphere.security.doc/GUID-3BB93F2C-3872-4F15-AEA9-90DEDB6EA145.html)