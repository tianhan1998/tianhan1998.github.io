---


layout:     post
title:      "在公网环境WLS2中使用Docker搭建RTMP服务器运行自己的直播平台"
subtitle:   "HLS"
date:       2021-08-21
author:     "Sakura"
tags:
    - 项目
    - 折腾
    - 代码
    - 心得
typora-root-url: ..
---
[toc]

# 前言（废话）

终于从运营商那里拿到了公网IP，准备折腾原来搞不了的东西。（河南联通打个电话就能要到，早知道早点要了）。

首先是打算把原来的网盘重写一下（大二时期用SSM写的，写的十分丑陋，自己都不想改了），重写过程中，又想着把写代码过程直播一下，遂又想到自己之前搞的直播平台，平台搞好只不过瓶颈在阿里云带宽上（只买了1M的轻量级云服务器，测试720p 1000码率 24fps 顿卡，画面不能再降，再降就看不下去了）。就把网盘先停了，用一天时间把直播服务器转移到本地。

# 准备

## 软件

要自己推流你需要准备两个工程

1. RTMP推流服务器
2. 直播间或播放器

## 硬件

公网IP（这算硬件？），或高带宽流量的云服务器（钞能力解决）

公网IP没有的找当地ISP要，据说大城市、电信联通好要到。

# 配置

## RTMP推流服务器

通过Docker容器，快速部署RTMP-Nginx服务器，在本地公网IP环境中部署，由于本机一般都是Windows系统，而Windows装Docker则需要虚拟机或WSL。本文通过WSL来运行Docker。

### 安装WSL

#### 1.BIOS打开虚拟化

打开后在任务管理器中的性能选项卡中可以看到开启![虚拟化](/img\wsl2\虚拟化.png)

#### 2.微软商店安装Ubuntu

![虚拟化](/img\wsl2\3.png)

安装完后运行可能遇到以下问题

![](/img/wsl2/4png.png)

或者在WSL1升级WSL2过程中可能遇到

**由于虚拟磁盘系统限制，无法完成请求的操作。虚拟硬盘文件必须是未压缩和未加密的文件，并且不能是稀疏文件。**

这两种问题用一种方法解决，找到Windows的如下路径

```
C:\Users\${username}\AppData\Local\Packages\CanonicalGroupLimited.xxxxxxx
```

最后一个文件夹是你安装的Ubuntu版本

找到文件夹下的LocalState文件夹，属性，高级

<img src="/img/wsl2/image-20210822144049170.png" alt="image-20210822144049170" style="zoom:80%;" />

找到**压缩内容以便节省磁盘空间**去掉这个勾即可解决

### WSL2安装Docker

#### 1.更换国内源

```shell
vim /etc/apt/sources.list
```

替换为以下内容

```shell
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal universe
deb http://mirrors.aliyun.com/ubuntu/ focal-updates universe
deb http://mirrors.aliyun.com/ubuntu/ focal multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal-security universe
deb http://mirrors.aliyun.com/ubuntu/ focal-security multiverse

```

#### 2.添加Docker源

依次执行下列命令

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt update

```

#### 3.安装 Docker

```shell
sudo apt install -y docker-ce
```

安装完成后通过service命令启动Docker

```shell
sudo service docker start
```

### 运行Nginx-RTMP

使用**alfg/nginx-rtmp**容器搭建服务器

Docker拉取镜像

```shell
docker pull alfg/nginx-rtmp
```

运行

```shell
sudo docker run -d -p 1935:1935 -p 5000:80 --name rtmp --restart=always alfg/nginx-rtmp
```

1935端口为RTMP推流端口 第二个映射端口为HLS流地址

访问宿主机5000端口检查Nginx运行情况（宿主机与WSL2公用一个localhost）

![image-20210822152402219](/img/wsl2/image-20210822152402219.png)

可以访问

至此，RTMP服务器搭建完毕 

## 直播间

可以使用VLC播放器或网页HTML来播放RTMP，本文主要讲解HTML方式

### 协议

现在的直播协议比较常用的就是RTMP，延迟很小，大部分直播平台都使用的是他，不过有一个小问题，RTMP是Adobe公司开发的协议。只支持自家Flash播放器（html5**<video>**标签不支持)。现在都1202年了，大部分浏览器都放弃了对Flash的支持。

我们可以通过HLS来曲线救国

> **HTTP Live Streaming**（缩写是**HLS**）是一个由苹果公司提出的基于HTTP的流媒体网络传输协议。是苹果公司QuickTime  X和iPhone软件系统的一部分。它的工作原理是把整个流分成一个个小的基于HTTP的文件来下载，每次只下载一些。当媒体流正在播放时，客户端可以选择从许多不同的备用源中以不同的速率下载同样的资源，允许流媒体会话适应不同的数据速率。在开始一个流媒体会话时，客户端会下载一个包含元数据的extended M3U (m3u8)playlist文件，用于寻找可用的媒体流。
>
> HLS只请求基本的HTTP报文，与实时传输协议（RTP）不同，HLS可以穿过任何允许HTTP数据通过的防火墙或者代理服务器。它也很容易使用内容分发网络来传输媒体流。
>
> —— [摘自维基百科](https://link.jianshu.com?t=https%3A%2F%2Fzh.wikipedia.org%2Fwiki%2FHTTP_Live_Streaming)

### 前端配置

现在的常见做法是通过ffmpeg将RTMP转为HLS(m3u8) 再通过Nginx服务器以HTTP协议获取HLS流。HTML5部分浏览器原生支持，比如Safari、edge。遇到不支持的软件通过引入衍生开源项目来补充。例如video.js

```html
<link href="http://cdn.bootcss.com/video.js/6.0.0-RC.5/alt/video-js-cdn.min.css"rel="stylesheet"/>
<script src="http://cdn.bootcss.com/video.js/6.0.0-RC.5/video.js"></script>
<script src="http://cdn.bootcss.com/videojs-contrib-hls/5.3.3/videojs-contrib-hls.js"></script>
<script>
            var player = videojs("hls-video");
            player.play()
 </script>
```

```html
<video>
        id="hls-video"
        width="1280"
        height="720"
        class="video-js vjs-default-skin"
        playsinline
        webkit-playsinline
        autoplay
        controls
        preload="auto"
        x-webkit-airplay="true"
        x5-video-player-fullscreen="true"
        x5-video-player-typ="h5">
    <!--xxxx部分为你的HLS地址 -->
    <source src="http://xxxxxxxxxxxx/xxxx.m3u8"
            type="application/x-mpegURL"/>
</video>
```

### 推流

打开OBS，设置推流地址

```
rtmp://localhost:1935/stream
```

串流秘钥为空

![image-20210822152053008](/img/wsl2/image-20210822152053008.png)

开始推流，无问题![image-20210822152122452](/img/wsl2/image-20210822152122452.png)

运行你自己的直播间项目

![image-20210822153342635](/img/wsl2/image-20210822153342635.png)

内网宿主机访问成功

## 公网访问

运行成功后，我们尝试在公网访问项目。

访问线路大概是这样

**域名**——*DNS解析*——>**公网IP**——*路由器DMZ*——>**内网宿主机**——*windows端口转发*——>**WSL**

### DMZ主机映射内网宿主机

首先在路由器中设置DMZ主机，

![image-20210822153952517](/img/wsl2/image-20210822153952517.png)

设置后，我们就可以通过域名或自己的公网IP来访问到宿主机。但是现在还访问不到WSL。

虽然宿主机与WSL之间公用一个localhost，可是这仅限在本机中可以访问，在公网中向宿主机端口发送请求是访问不到WSL的。

![image-20210822155350261](/img/wsl2/image-20210822155350261.png)

192.168.0.253为局域网中宿主机IP，可以看到只有宿主机自己才可以直接访问虚拟机

### windows端口转发

WSL2实际上是在Hyper-V上运行的一个虚拟机，与宿主机是在不同的网段 。

在WSL中输入ifconfig

![image-20210822160156823](/img/wsl2/image-20210822160156823.png)

eth0为虚拟机IP，可以发现，与宿主机的192.168.0.253不在同一网段

宿主机直接访问该IP下的Docker容器，可以接通

![image-20210822160313655](/img/wsl2/image-20210822160313655.png)

所以我们现在要做的是将访问宿主机（192.168.0.253）的请求转发到虚拟机（172.25.157.124）上

我们通过windows自带的端口转发来实现，管理员CMD中运行

```shell
netsh interface portproxy add v4tov4 listenport=5001 connectaddress=172.25.157.124 connectport=5000 protocol=tcp
```

**listenport为监听的端口号**

**connectaddress为转发目标IP**

**connectport为转发目标端口**

**protocol为协议**

我们将connectaddress设定为虚拟机IP（虚拟机中ifconfig eth0的地址），connectport设定为对应Docker容器的端口号5000

由于我们在宿主机上直接用localhost推流，所以RTMP的1935端口不用进行转发（如果你需要在外网或局域网其他电脑进行推流的话，也要对1935端口进行转发）

**注意！！！！listenport端口不能和connectport端口一样，否则外网访问不到虚拟机（也就是说不能监听5000再转发到虚拟机的5000端口，不知道什么原理，在我这是这样的。**

此外我们可以通过下面的命令来查看windows自带的端口转发

```shell
netsh interface portproxy show all
```

![image-20210822161658169](/img/wsl2/image-20210822161658169.png)

通过下面的命令来清空端口转发

```
sudo netsh interface portproxy reset
```

在设置过端口转发后我们测试联通情况

![image-20210822161947668](/img/wsl2/image-20210822161947668.png)

可以通过局域网宿主机IP来访问虚拟机的Docker容器了（内网访问成功）

注意是5001端口，这是宿主机监听的端口

![image-20210822162117360](/img/wsl2/image-20210822162117360.png)

公网IP和域名访问成功（外网访问成功）

**注意**

**如果你设置的监听端口号和连接端口号一致，比如监听5000转发5000**

**那么应该是内网通但外网不通（内网可以通过192.168.0.253访问到虚拟机，但是公网IP和域名访问不到）**

### 固定WSL的IP

在Hyper-V上运行的Linux不仅网段跟宿主机不一致，而且每次重新运行都会获取新的IP，这非常的不方便，难道每次重启WSL都要重新配置windows端口映射，那多麻烦。

解决方法是通过脚本，在WSL写一个脚本，每次启动将自己的IP通过挂载盘写入到宿主机的hosts中，在hosts中固定一个域名，做一个类似DDNS的功能，windows端口转发固定转发到域名就好。

GitHub上好像有一个go-wls2开源项目，我用着无效。。还是自己写吧

```shell
vim /etc/init.wsl
```

写入以下内容

```
ip=`ifconfig eth0 | grep inet | awk 'NR==1{print $2}'`
sed -i '/.* wsl.local/d' /mnt/c/windows/System32/drivers/etc/hosts
echo "$ip wsl.local" >> /mnt/c/windows/System32/drivers/etc/hosts
```

此脚本将WSL IP写入宿主机hosts，并设定域名为wsl.local

所以在windows端口转发，就只用配置下面的命令就可

```shell
netsh interface portproxy add v4tov4 listenport=5001 connectaddress=wsl.local connectport=5000
```

这样配置在WSL重启后windows端口转发配置也不用变化，脚本在hosts中配置好了新IP。

