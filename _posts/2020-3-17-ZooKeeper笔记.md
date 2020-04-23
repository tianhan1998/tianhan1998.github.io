---
layout:     post
title:      "ZooKeeper学习笔记"
subtitle:   "\"动物管理者\",与Dubbo结合即可形成一个基本的分布式系统"
date:       2020-3-17
author:     "TH"
header-img: ""
tags:
    - 学习
    - ZooKeeper
    - 心得
---

1. zookeeper：一个领导者（Leader）和多个跟随着（Follower）组成的**服务器集群**
2. **集群有*半数以上*的节点存活，就可以正常服务**
3. 全局数据一致，每个服务器数据一样！！
4. 更新请求顺序进行，每个客户端的更新请求按照发送顺序来执行
5. 数据更新原子性：要么成功要么失败
6. 实时性，在一个时间范围内，能读到最新数据（不同服务器同步速度快）

## 数据结构

树形结构：如Linux ntfs文件系统与**Unix文件系统很相似**

节点叫ZNode默认存储**1MB**的数据，每个ZNode都可以**通过其路径唯一标识**（每个节点有唯一标识，比如路径/Znode1/Znode2)

## 应用场景

![image-20200317131200620](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317131200620.png)

### 统一命名服务：

![image-20200317131343327](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317131343327.png)

访问域名而不是服务器

### 统一配置管理：

1. 要求每个节点配置信息一致
2. 配置文件修改后快速同步

ZooKeeper实现：

把配置文件写到一个ZNode上

各个客户端服务器都监听，一个更新，全部同步

一旦ZNode修改，通知各个客户端

### 统一集群管理

分布式中实时掌握各个节点的状态

ZooKeeper实现：

节点信息写入一个ZNode

监听这个ZNode

### 服务器动态上下线

客户端要看到服务器上下线的变化

可以迅速看到某个服务器挂掉或变化

### 软负载均衡

让访问数最少的服务器处理最新客户端请求

![image-20200317132049886](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317132049886.png)

## 配置参数

1.tickTime=2000 心跳数，单位ms，超过这么长时间，断线

2.initLimit=10 初始化通信连接容忍度 单位tick心跳，10x2s=20s等待容忍时间

3.syncLimit=5 同步容忍时间 单位tick心跳 5x2s=10s 同步等待10秒容忍

4.dataDir数据持久化路径

5.clientPort客户端端口

# 内部原理

由于半数机制（超过半数台机器存活才可用）适合安装在**奇数**台服务器

例如5台机器 5/2>2  3台在工作即可运行 可容忍2台挂机

4台机器 4/2=2>台 只可容忍1台挂机

## 选举Leader 重点！

### 半数选举机制（启动时）

每个Leader候选人先投自己，如果超不过半数票则投给最大的myId服务器

选出Leader后再次加入新服务器，Leader不改变

## 节点类型

两种：持久或者短暂性

持久：断开连接节点不删除(Persistent)

短暂：断开连接即刻删除自己(Ephemeral)

1.持久化目录节点：断开连接后节点依旧存在

2.持久化顺序编号目录节点：断开连接后节点依旧存在，并且按照顺序编号

编号单调递增，由父节点维护

可以通过顺序号来推断事件的顺序

3.临时目录节点：断开连接后，节点被删除

4.临时顺序编号目录节点：断开连接时被删除，创建时按照顺序标号单调递增

## Stat结构体

![image-20200317162638793](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317162638793.png)

## 监听器原理（重点）！！

1.首先有一个main()线程

2.在main线程创建ZooKeeper客户端

会创建两个线程 一个负责通信（connect）一个负责监听（listener）

3.通过connect线程将注册的监听事件发送给ZooKeeper

4.在ZooKeeper的注册监听器列表中将注册的监听事件添加到列表里

5.ZooKeeper监听到数据或路径变化就会把这个消息发送给listener线程

6.listener线程内部调用了process()方法

## 写数据流程

客户端向一个服务器发送一个写请求

如果他不是Leader，会把信息转发给Leader

Leader广播写请求，所有Server写成功后通知Leader

当Leader收到了**大于一半**的成功了，那么数据就算成功了

之后Server1就会通知客户端认为写成功了

![image-20200317163403723](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317163403723.png)



# 实战

## 分布式安装部署

### zkData创建myid文件，编写对应编号

### 配置zoo.cfg

![image-20200317160223023](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317160223023.png)

server. A=B:C:D

A 服务器编号

B 服务器ip地址

C 与Leader信息交换的端口号

D Leader挂了之后选举信息端口号

启动：bin/zkServer.sh start

客户端不加start

bin/zkCli

## 命令行操作

1. ls [path] 

   ![image-20200317160954927](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317160954927.png)

2. ls2 [path] 详细数据

   ![image-20200317161011930](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161011930.png)

3. 创建节点

   create [path] [data]

   ![image-20200317161317946](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161317946.png)

   ![image-20200317161350066](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161350066.png)

4. 取数据

   get [path]

   ![image-20200317161501640](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161501640.png)

5. 创建短暂节点(断开客户端连接后删除)

   create **-e** [path] [data] 

   ![image-20200317161626942](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161626942.png)

6. 创建序号节点

   create **-s** [path] [data]

   ![](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161812621.png)

7. 修改节点值

   set [path] [data]

   ![image-20200317161926151](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317161926151.png)

8. 节点值变化监听

   一次性有效

   get [path] watch

9. 监听子节点变化（路径变化）

   ls [path] watch

10. 删除节点

    delete [path]

    ![image-20200317162442759](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317162442759.png)

11. 递归删除节点

    deleteall [path]或者rmr(过时)

## API应用

### 添加pom文件

![image-20200317163500973](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317163500973.png)

### 拷贝log4j

![image-20200317163614958](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317163614958.png)

### 连接ZooKeeper

![image-20200317164135480](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317164135480.png)

### 创建子节点

![image-20200317164614773](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317164614773.png)

create(path,data,ACL权限,创建模式（短暂或序号）)

### 获取子节点并监控

![image-20200317165253905](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317165253905.png)

![image-20200317165130016](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317165130016.png)

因为需要watch所以写到监听器内部类中

### 判断节点是否存在

![image-20200317165530454](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317165530454.png)

### 监听服务器动态上下线

#### 服务器向ZooKeeper写入自己信息

![image-20200317170956459](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317170956459.png)

![image-20200317171029003](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317171029003.png)

![image-20200317171108607](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317171108607.png)

![image-20200317171039075](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317171039075.png)

#### 客户端读取服务器信息

![image-20200317171918530](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317171918530.png)

![image-20200317171928834](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317171928834.png)

![image-20200317172002497](C:\Users\tianh\AppData\Roaming\Typora\typora-user-images\image-20200317172002497.png)

# 企业面试真题

## 选举机制

半数机制，达到半数，选择其作为Leader

## 监听原理

main中包含listener和connect

connect注册监听器，监听器列表增加一个客户端

当满足监听器要求，回调listener线程process()方法

服务器返回数据

## 部署方式 角色有哪些 集群最少需要几台机器

单机和集群

leader和follower

3

常见命令

get set create delete ls ls2