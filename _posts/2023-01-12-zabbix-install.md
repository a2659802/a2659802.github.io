---
layout: post
title: zabbix部署
date: 2023-01-12
categories: blog
tags: [zabbix,linux]
description: 在debian11部署zabbix
---

## 步骤

1.  安装

```bash
# init repository
 wget https://repo.zabbix.com/zabbix/5.0/debian/pool/main/z/zabbix-release/zabbix-release_5.0-2%2Bdebian11_all.deb
 dpkg -i zabbix-release_5.0-2+debian11_all.deb
 apt update

# install
 apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-agent
```

2. 创建数据库
```bash
# 如果没有安装mysql
# apt install mariadb-server 
mysql -uroot

mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user zabbix@localhost identified by 'zabbix';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```

3. 初始化数据库
`zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -pzabbix zabbix`

4. 开启之前临时关闭的限制
```bash 
# mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

5. 配置
```
编辑配置文件 /etc/zabbix/zabbix_server.conf
DBPassword=password

编辑配置文件 /etc/zabbix/nginx.conf 
listen 80; 
server_name 192.168.88.10; # example.com 改为部署机器的ip

编辑时区 /etc/zabbix/php-fpm.conf
php_value[date.timezone] = Asia/Shanghai
```

启动
`systemctl restart zabbix-server zabbix-agent nginx php7.4-fpm`

如果失败，大概率是nginx还有别的配置，自行查找


## 参考
- [官方文档](https://www.zabbix.com/cn/download?zabbix=5.0&os_distribution=debian&os_version=11&components=server_frontend_agent&db=mysql&ws=nginx)
- [Zabbix5.0+Ubuntu18.04+Mysql+Nginx安装部署](https://blog.csdn.net/ITerated/article/details/125385358)

