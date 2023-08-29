---


layout:     post
title:      "Eclipse导入Idea的Maven项目所需的配置踩坑"
subtitle:   "Eclipse不好用。。"
date:       2023-08-29
author:     "Sakura"
tags:
    - 折腾
    - 兼容
typora-root-url: ..
---
# 前言

最近接了一个老项目，SSM+JSP架构的，甲方直接扔了一个Eclipse的项目模板过来。。我们在Idea里导入开干，干完后因为是项目是从Eclipse创建的所以直接发回去了。。结果甲方打不开。我这边就又用Idea的Eclipse格式导出，导出之后保险起见也下了个Eclipse试了下，踩了不少坑才能运行，这边记录一哈。多说一嘴，回到了SSM才知道SpringBoot真的友好，真的省了不少配置事。

# 折腾

首先用Idea里的Export

![image-20230829152739138](/img/eclipse-1.png)

![CleanShot 2023-08-29 at 15.34.01](/img/eclipse-2.png)

确定后发现.classpath和.project有变化，这时导入还不行，需要做一些配置。

## Settings.xml

Eclipse的Preferences Maven->User Settings配置settings.xml文件。

![eclipse-3](/img/eclipse-3.png)

## Tomcat配置

![CleanShot 2023-08-29 at 15.50.22](/img/eclipse-4.png)

没有Server配置不了Tomcat的需要在Eclipse的Help->Eclipse Marketplace安装tomcat插件，支持Tomcat9，网上的老解决方法用Kepler一般只支持到8。

![CleanShot 2023-08-29 at 16.02.25](/img/eclipse-5.png)

## 项目配置（重点）

配置好tomcat大概也没法运行，首先module就检测不到web，没法部署，按照下面方式修改配置

![CleanShot 2023-08-29 at 15.44.38](/img/eclipse-6.png)

Dynamic Web Module改成3.1，忘记了是不是修改这里就有了module，后面还需要改别的，这时候应该是有module但是部署过去什么都没有，404。因为没有把编译好的文件发布到tomcat

![CleanShot 2023-08-29 at 17.51.20](/img/eclipse-7.png)

在Deployment Assembly中如上图配置好，意思是将classes文件和maven lib发布到对应tomcat webapps目录下。

另外在Web Project Settings下可能有Bug，Context root一直为空，设置任何值后保存也为空。说明项目配置不太规范，解决方法是在Project Natures增加Web Properties

![CleanShot 2023-08-29 at 17.54.31](/img/eclipse-8.png)

在pom.xml右键，点Run as里的install编译（没有compile，懒得弄了直接install）

![image-20230829175842523](/img/eclipse-9.png)

install也可能碰到许多问题，这里是根据项目走的，比如我就遇到了编码不对，可以在项目Properties的Resource里把编码改成GBK（前提是你项目是GBK的编码）来解决。又或者发到甲方那边报没有javax.annoation Class的错误，看了一下是我的jdk里自带javax.annoation（用的JDK是Amazon Corretto 8），甲方那边电脑jdk没有。于是maven里加上就行了，注意修改pom一定要ctrl+s保存后再update project，否则不会应用，无语了。

最后在甲方电脑上还有个问题愣是解决不了，编译好的classes文件愣是不发布到tomcat的webapps目录里。Deployment Assembly已经设定完了但还是不复制，我自己的电脑上是可以发布过去的，最后自己手动复制过去直接就能运行了。但这样甲方应该是改不了代码了，除非他也手动把编译好到classes复制到webapps目录那边。。。

其实在帮甲方部署的时候还遇到了好多Bug，例如tomcat启动不了，解决方法是要修改用户权限，还有项目属性里点击Apply and close就死机（我这边也会死机，很怪，我用的eclipse是2021-12，甲方那边是2021-09，可能是Idea转换项目问题）。以及tomcat缓存问题。。各种小问题一堆，只能一个一个去搜。感觉没收钱挺麻的。下次得给甲方提前订好，这种运维也要收点钱才行TAT。
