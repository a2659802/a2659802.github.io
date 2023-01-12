---
layout: post
title: linux passwd自动化改密方案
date: 2023-01-06
categories: blog
tags: [linux]
description: 控制标准输入模拟手动输入新旧密码的过程来实现passwd自动化
---


## linux passwd自动化改密方案
对于普通用户而言，可以直接使用passwd来改密，但是需要通过交互式命令行来输入新旧密码，
如何将其变成非交互式以便自动化呢？这里有bash和python两种参考方案

不管哪种方式，基本原理都是往标准输入里面按顺序输入密码。
但是，对于普通用户和root账户，它们的输入要求略有不同
对于root账号，执行passwd修改自身密码时，不需要输入旧密码，并且不受密码规则约束
但是对于普通账户，执行passwd修改自身密码时，需要先输入旧密码，受密码规则约束

下面的脚本方案和Python方案都是针对非root用户的，root用户需要稍微改以下具体实现。
如果有切换自(su)账号的改密，那么脚本的实现会更加困难

### 纯脚本(bash)
需要先安装sshpass实现非交互式登陆，
为了跳过确认 Are you sure you want to continue connecting (yes/no/[fingerprint])?
需要设置参数-o StrictHostKeyChecking=no

```bash
#!/bin/bash
# 假设旧密码为1234 新密码为T2659802com23
# 下方文件ppp可以用随机字符串代替

echo -e "1234\nT2659802com23\nT2659802com23\n" > /tmp/ppp
sshpass -p '1234' ssh test@10.1.1.32 -o StrictHostKeyChecking=no passwd < /tmp/ppp
rm -f /tmp/ppp

# 执行成功，有两个识别点
# 1. 返回值为0
# 2. 标准输出有关键字 successfully

# 执行失败，两次密码不同
# 1. 返回值1 
# 2. 标准输出有关键字 do not match

# 执行失败，密码是回文(aaaaaaaa)
# 1. 返回值1
# 2. 关键字 palindrome

# 执行失败，密码与旧密码相似(qwer)
# 1. 返回值1
# 2. 关键字 similar to

# 执行失败，密码短于要求(#!%qa)
# 1. 返回值1
# 2. 关键字 shorter than

# 执行失败，旧密码错误（这个一般不会出现，因为如果旧密码不对，登陆那一步就会先报别的错误）
# 1. 返回值1
# 2. 关键字 Authentication token manipulation error

# 登陆失败，密码错误
# 1. 返回值5
# 2. 关键字 Permission denied

# 执行失败，密码不符合自定义约束(qwerasdf)
# 1. 返回值1
# 2. 关键字 dictionary check 

# 登陆失败，连接失败
# 1. 返回值255
# 2. 关键字 Connection refused

# 如果需要存储原始的错误信息，建议找到关键字BAD PASSWORD，将后面的字符串直到换行作为原始输出
# 因为标准输出还混有别的信息，这个自行尝试一下就知道了

# 然后系统安装时的语言会影响标准输出的内容，这个暂时还没有时间去测试
```
### 问题
1. 中文系统sshpass无法使用，因为该程序无法处理中文的交互式登陆
2. 返回值不同系统可能不同，实际测试中，centos7修改失败的返回值都是1，ubuntu17.10返回值都是10
3. 关键字确实受对端语言的影响，而且翻译并不完全（中英夹杂），所以有时候如果不能识别到关键字，建议把标准错误输出都记录下来

### python
如果使用python，可以借助第三方库paramiko来实现ssh登陆，不需要sshpass
用python的好处是有些错误处理起来比较方便，例如连接、登陆阶段的错误

以下是python3的示范，python2也差不多
```python 
import paramiko

def change_password(ip,port,username,oldpwd,newpwd):
	ssh = paramiko.SSHClient()
	# 跳过确认host key
	ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
	try:
		ssh.connect(hostname=ip, port=port, username=username, password=oldpwd)
	except Exception as e:
		# 登陆失败，这里有几种错误：连接方面、用户密码方面
		# 需要根据Exception的类型来判断失败原因
		return False
		
	stdin,stdout,stderr = ssh.exec_command('passwd')
	# 输入3次密码
	stdin.write('%s\n' % oldpwd)
	stdin.write('%s\n' % newpwd)
	stdin.write('%s\n' % newpwd)
	stdin.flush()
	# 发送EOF, 这是因为改密失败时，会要求重新输入，这时可以通过EOF来终止
	stdin.channel.shutdown_write()
	
	# 命令返回值
	exit_code=stdout.channel.recv_exit_status()
	print('exit with:%d\n' % exit_code)
	
	# 如果需要提取关键字，一般是从stderr里面提取
	print('stdout:%s\n' % stdout.read().decode())
	print('stderr:%s\n' % stderr.read().decode())
	
change_password('10.1.1.32',22,'test','1234','aaaa')
```

## usermod方案
如果把限制放宽，只通过特权账号来修改密码，那么可以使用usermod命令来实现。
这也是ansible的user模块的实现原理。
由于usermod本身就支持非交互式改密，而且网上方案也很多，这里就不做细节讨论，仅提供一个方向。