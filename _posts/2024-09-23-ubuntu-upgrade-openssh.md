---
layout: post
title: ubuntu升级openssh版本
date: 2024-09-23
categories: blog
tags: [openssh, ssh]
description: 通过升级ssh版本解决被漏洞扫描工具报告的高危漏洞
---

## ubuntu升级openssh版本

某天收到反馈说openssh存在漏洞，要求修复
```
命令注入漏洞(CVE-2020-15778)
输入验证错误漏洞(CVE-2020-12062)
QpenSSH 安全漏洞(CVE-2021-28041)
OpenSSH 安全漏洞(CVE-2021-41617)
OpenSSHssh-agent 远程代码执行漏洞(CVE-2023-38408)
OpenSSH 信息泄漏漏洞(CVE-2023-51385)
OpenSSH 安全漏洞(CVE-2016-20012)
OpenSSH信息泄露漏洞(CVE-2020-14145)  
OpenSSH 安全漏洞(CVE-2023-51384)
安全漏洞(CVE-2023-51767)
QpenSSH 安全漏洞(CVE-2023-48795)
OpenSSH 授权问题漏洞(CVE-2021-36368)
```

修复思路：
看了一圈其实都不算真正意义的漏洞，估计检测工具是通过版本来判断的，故修复方向应该为升级openssh的版本。

### 过程

1. 去[官网](https://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/)下载最新版本，当时选的是9.8
2. 安装依赖
```bash
apt install -y gcc build-essential manpages-dev make perl zlib1g zlib1g-dev libssl-dev linux-libc-dev libpam0g-dev
```
3. 解压编译
```
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-zlib --with-privsep-path=/run/sshd --with-pam  --with-md5-passwords

make -j
```
4. 编辑Makefile，修改DEST_DIR为目标系统根目录（如果是本机则不需要修改），然后执行`make install`

#### 补丁制作

1. 创建一个目录，例如/tmp/fakeroot
2. 修改DEST_DIR然后`make install`
3. `cd /tmp/fakeroot` 然后执行`tar -czvf patch.tar.gz *`打包
4. 在目标系统执行`tar -zxvf patch.tar.gz -C /`解压到根目录

#### ISO制作

1. 用`xorriso -osirrox on -indev "stdio:$iso_path" -extract / "$iso_tmpdir"`解压ISO
2. 用`unsquashfs -d "$squashfs_dir" "$tmpdir"/filesystem.squashfs`解压文件系统
3. 修改nocloud/user-data的内容，在late-commands里增加
```yaml
autoinstall:
  late-commands:
    - tar zxvf /target/home/ssh9.tar.gz -C /target
    - rm /target/home/ssh9.tar.gz
```
4. 用`mksquashfs  "$squashfs_path" "$ubuntu_squashfs_path" -comp xz -b 1M -noappend`压缩文件系统
5. 压缩ISO
```
 xorriso -as mkisofs -r \
    -V "SRHINO_AMD64" \
    -c isolinux/boot.cat \
    -b isolinux/isolinux.bin \
    -no-emul-boot \
    -boot-load-size 4 \
    -boot-info-table \
    -eltorito-alt-boot \
    -e boot/grub/efi.img \
    -no-emul-boot \
    -isohybrid-gpt-basdat \
    -o "$iso_path" "$iso_tmpdir"
```

**注**

ubuntu20.04及其之后都使用cloud-init方式实现自动化安装，将自动安装参数autoinstall ds=nocloud;s=/cdrom/cloud_init/添加到isolinux/txt.cfg文件中quiet之前，然后将必要的文件user-data, meta-data复制到制定文件路径 /cloud_init目录下。在user-data和meta-data中即可开启自动安装设置，不设置即走默认的安装设置。

### 其他尝试

尝试编译deb包，但是遇到不少和系统相关的依赖问题不好解决，加上时间紧急，只好放弃


### 参考资料
- [ubuntu-openssh升级](https://www.cnblogs.com/subsea/p/17682962.html)
- [ubuntu nocloud-init](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
- [构建ssh deb包](https://gist.github.com/prbinu/2ec8fd91071f20eadfc5c87d20340c50)