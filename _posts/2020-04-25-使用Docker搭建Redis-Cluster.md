---
layout: post
title: 使用Docker搭建Redis-Cluster
subtitle: dockerda搭建redis集群
tags: [Docker, Redis, Redis-Cluster]
---

## 1.环境准备

1、3台centos7.0操作系统的虚拟机，IP分别为`192.168.100.40`，`192.168.100.41`，`192.168.100.42`，关闭`firewalld`，方便调试。

2、在每个操作系统中安装好Docker，下载redis的[redis.conf](https://github.com/redis/redis/blob/unstable/redis.conf)，需要修改如下配置（可以使用shell的for进行快速创建）

```shell
port 6379  #redis对外暴露的端口
requirepass 1234 #redis密码
masterauth 1234 #redis连接主节点的密码
protected-mode no #设置redis允许外部网络访问
daemonize no #守护进程模式，这里使用docker启动的方式，设置为no
appendonly yes #开启AOF
cluster-enabled yes #开启集群模式
cluster-config-file nodes.conf #集群模式下节点的配置信息
cluster-node-timeout 15000 #集群模式下的节点超时时间 一般redis会选取一批节点，将这些节点操过15000/2没ping的都进行ping/pong
cluster-announce-ip 192.168.10.11 #节点所在服务器的IP地址
cluster-announce-port 6379 #节点映射端口
cluster-announce-bus-port 16379 #节点总线端口
```

3、redis.conf文件需要对不同节点进行对应配置，这里每个配置文件的`requirepass`和`masterauth`都设置成一样的，否则会出现集群主从连接不上的问题。

4、创建对应文件目录，方便启动容器时将redis配置文件和数据挂载

```shell
#192.168.100.40上的文件目录
/usr/local/redis_docker/
├──6379
|	├──conf
|	|	└──redis.conf #外部挂载的配置文件
|	└──data #RDB数据存放在这里
└──6380
    ├──conf
    |	└──redis.conf
    └──data
#192.168.100.41上的文件目录
/usr/local/redis_docker/
├──6379
|	├──conf
|	|	└──redis.conf #外部挂载的配置文件
|	└──data #RDB数据存放在这里
└──6380
    ├──conf
    |	└──redis.conf
    └──data
#192.168.100.42上的文件目录
/usr/local/redis_docker/
├──6379
|	├──conf
|	|	└──redis.conf #外部挂载的配置文件
|	└──data #RDB数据存放在这里
└──6380
    ├──conf
    |	└──redis.conf
    └──data
```



------

## 2.创建redis容器

1、每台服务器执行相对对应的命令

```shell
docker run -it -d --restart=always --name redis-6379 --net host \ #这里的--net host一定要注意，docker默认是桥接模式，不用这个--net host会访问不到，redis官方文档已写出
  -v /usr/local/redis_docker/6379/conf/redis.conf:/usr/local/etc/redis/redis.conf \
  -v /usr/local/redis_docker/6379/data:/data \
  redis redis-server /usr/local/etc/redis/redis.conf
```

2、`docker ps`检查容器是否成功启动，有问题`docker logs 容器ID`查看启动日志排错

## 3.创建 Redis-Cluster 集群

1、进入某个容器创建集群

```
# 进入容器
docker exec -it redis-6379 /bin/bash
# 切换至指定目录
cd /usr/local/bin/
#关联集群
redis-cli -a 1234 --cluster create 192.168.100.40:6379 192.168.100.40:6380 192.168.100.41:6379 192.168.100.41:6380 192.168.100.42:6379 192.168.100.42:6380 --cluster-replicas 1
```

按照控制台的指示逐步操作。成功后可以使用`cluster info`查看集群状态，state为ok就是正常的，之后就可以测试集群了。

如果需要查看集群的槽点分配信息，可以进入宿主机下创建的data文件下查看，RDB文件也在这下面，如果开启了AOF，AOF文件也在这下面。

## 4.集群使用的注意事项

- 集群中默认是使用db0的，不支持多库
- 因为key的槽点分配，所以在使用是要注意mset等批处理操作的key的槽点要在一台机器。
- 主节点超过一半挂了或者某个槽点的服务不可用了，集群就不可用