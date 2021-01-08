---
layout: post
title: Linux下部署Java应用程序防火墙设置
subtitle: Linux基础
tags: [Linux, Java, 防火墙, 端口]
---

## 1. 关闭防火墙（一劳永逸）

```shell
#永久关闭防火墙
systemctl enable firewalld
#零时关闭防火墙
systemctl stop firewalld
```

## 2.开放防火墙端口

```shell
#查看防火墙开放端口列表
firewall-cmd --list-port
#开放9009防火墙端口
firewall-cmd --zone=public --add-port=9009/tcp --permanent
#其中的--add为新增，--query为查询，--remove为移除
```

