---
layout:     post
title:      "基于SSM实现的小网盘"
subtitle:   "麻雀虽小五脏俱全"
date:       2020-05-04
author:     "TH"
header-img: "img/OnlineDisk.jpg"
tags:
    - 项目
    - SSM
    - 代码
    - 心得
---
# OnlineDisk
一个小网盘
# 记录

## 第一天
重新配置ssm都花了半天，各种莫名其妙的问题，什么mybatis的mapper映射不上去，静态资源拦截等等，重新弄得我头大，我按照原来的demo一点点改，最后代码基本上都改成老的了，还是有错。最后直接新建新项目然后拉老的demo复制粘贴。所以代码里叫 **NewOnlineDisk**。idea新建的maven项目结构不能乱动，每次修改之后再reimport就会变回去。所以直接用默认的配置就好,静态资源拦截记得在是spring-mvc.xml里添加一个<mvc:default-servlet-handler/>就好。我的导入不进去实际是因为jsp里面引入js文件使用了相对路径，改成动态获取绝对路径就行了。

***
### 第一天实现
- 项目的登录和注册（包括cookie保持登陆状态）
- 文件的上传和下载
- 前端页面基本完成
***
另外第一天最后push的时候还出了一点小插曲，因为前端框架文件太多，我idea一开始忘记取消代码检查了，速度极慢，最后还漏文件没有push上来。重新push的时候总是push不上去，上网查新建了新的分支再push，最后merge入主分支就解决了。因为之前我写 **ojtools** 这个项目的时候就不小心给我写完的代码全部搞丢了，最后用了 **Revert** 这个命令恢复的，所以现在关于git的东西都比较小心，害怕一不小心把代码全部报销= =
## 第二天
第二天主要任务是增加了删除文件功能，另外对上传和下载进行了优化，之前会被人用url换参数偷东西，现在会验证好名字并用id而不是 ~~filename，username~~ 来下载指定文件，比较科学。
***
### 第二天实现
- 文件的删除功能
- 上传下载的重写
- 前端页面修正
***
今天加的功能不多，但是代码量比第一天要多。首先给删除使用了事务，删除数据库上你存的文件记录，并删除硬盘上实际对应的文件，如果文件删除失败，那么要回滚数据库，把删除的记录撤销。事务我是用的在类上的@Transactional注解来实现。一开始在controller层的事务一直失效，最后上网查才知道其实是在xml中配置的扫描事务注解 <tx:annotation-driven>的问题，一开始这个配置是在applicationContext.xml里的，但controller的注解扫描是在spring-mvc.xml里，所以不仅ApplicationContext.xml里面要配置，spring-mvc.xml也需要配置，因为这个是根据上下文中的注解扫描实现的。网上原文是这样的
> 一般而言，事务都是加在Service层的，也可以加在Controller层

> 在spring-framework-reference.pdf文档中有这样一段话：
> > <tx:annotation-driven/>only looks for @Transactional on beans in the same application context it is defined in. This means that, if you put
> >  **<tx:annotation-driven/>**
> > in a WebApplicationContext for a DispatcherServlet, it only checks for @Transactional beans in your controllers, and not your services. 

> 这句话的意思是<tx:annoation-driven/>只会查找和它在相同的应用上下文件中定义的bean上面的@Transactional注解，如果你把它放在Dispatcher的应用上下文中,它只检查控制器上的@Transactional注解，而不是你services上的@Transactional注解。于是，我将事务配置定义在Spring MVC的应用上下文(spring-mvc.xml)中。

除此之外，还有不少新发现，学习了不少知识，比如
### 输入输出流关闭
在删除文件时，发现返回false删除失败，上网查阅资料有几种可能性
> 1. 此文件被使用的时候无法删除（比如网络输出没关闭流）
> 2. 判断此文件是否存在再做删除（exists）
> 3. 删除文件夹之前先删除文件夹下的所有文件（递归解决）
> 4. 判断是否删除成功会有返回值，文件名错了的话，删除文件不会报错。（new File("x://123.txt"),但是123.txt不存在，不报错

我的情况是第一种，在使用下载功能后，再进行删除的话就会失败。所以我去查看下载里是否关流。我实际上添加了流的关闭代码，但是不严谨，没有写到try-catch块里，我规范了一下代码，在try-catch块之前声明流，在块内赋值，finally内部关流。最后发现输出流没有关闭，报错

**java.io.IOException: 你的主机中的软件中止了一个已建立的连接。**

网上查这个Exception很广，查到了一个贴跟我相关，大概原因应该是我点击了下载按钮但没有确认下载，取消了下载框后导致流没有关闭。网上关于这个问题只有一个原因，而没有解决方案，最后我是靠try-with-resources的语法糖解决的，在try（）括号内声明你的流，在代码结束后会自动关闭，

``` 
try(BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(res.getOutputStream()))
```

### Spring-mvc中@PathVariable的参数丢失问题
如果使用@PathVariable来获取url内的参数，那么参数中'.'后面的东西会消失

比如我的参数是文件名xxxx.zip

那么获取到的参数filename=xxxx

'.'后面的东西没了，要解决这个，最简单的就是直接用@RequestParam，但是get的url略不美观

解决方法是这样，在@RequestMapping注解内修改，@RequestMapping("/DownLoad/{filename:.*}")

参数后面加上:.*就可，记住要重新部署项目才行，不能直接更新classes和resource，因为懒得等在这里卡了一会233

#### 总结下两天学到的东西
1. 静态资源导入要动态获取绝对路径
2. @Pathvariable会丢参数
3. @Transactional要放到不仅有扫描注解而且还要有扫描事务注解的xml对应的层上
4. try-with-resources语法糖不会关不上流
## 第三天
***
### 第三天实现
- ajax判断注册用户名是否重复
- 修复第一次访问/路径下会报500的bug（cookie没有判断null导致空指针）
***
现学现造的，虽然功能很简单，但由于我用js和jquery的时候非常少，所以这个简单功能写了好几个小时，主要就是前端js老是写错，然后还没有什么提示，就得猜。其次就是对jquery的不熟悉，要操纵label的话，不能用val修改，要用html。而且选择器的#老是忘。有点头疼
## 第四天
***
### 第四天实现
- 添加多任务上传
- 前端页面描述修改
- 在登录界面添加更新历史
- 修复可以上传空文件的bug
- 给上传文件添加了事务(虽然上传一般不出错)
***
第四天主要时间花在了前端，后端多文件上传很简单，新方法获取数组然后一个一个调之前的单上传就行了。事务也是注解然后添加一个手动回滚解决，主要是前端，我想添加一个更新列表，在bootstrap里面找了一会儿，也没找到合适的界面，最后还是用现在登陆的模板。制作人员那个360°动画还是挺好看的，我就想照着再做一个界面翻滚，照着已有代码修改，加新的选择器，新的事件监听，css属性修改就按照之前的函数，把对应的选择器名字修改下就好了，没有遇见什么难题。接下来准备搞一个留言功能，所有人可以在这里回复，然后用ajax实现异步刷新换页，明天应该能搞定
## 第五天
***
### 第五天实现
- 增加评论功能
- 增加了总上传文件大小上限
- 数据库更换时区
- 文件大小修改为MB显示
***
从第五天开始搞评论（留言？）功能，预计的需求就是所有人都能看到的留言，然后可以点赞，发出人可以删除。从最开始构思建表的时候就想了一会，点赞到底怎么实现，是在评论表里加上点赞id？还是新建一个表。最后上网查询正好发现csdn上有一个帖子询问点赞的数据库表。下面有一位大佬问了新浪的dba这个问题。收获许多，最后决定添加一个点赞表，表内有点赞用户和点赞评论id，可以查询这个表来判断是否点过赞。下面是帖子地址

<https://bbs.csdn.net/topics/391001223>

既然是评论，就要有评论时间，既然有时间，就会有格式转换，果然我又碰到了这个问题。一开始前端往后端转表单时间，时间的获取用jsp的usebean标签 new一个Date类型的bean，然后用fmt:formatDate来格式化。问题来了，一开始往后传数据，后台接到了之后，时间莫名增加了14个小时。感觉是时区的问题，上网查增加8小时、10小时、14小时的都有，解决方法就是在数据库url里把参数改成HongKong，问题解决。但改完之后数据传不到后台了，服务器报错

**Failed to convert value of type 'java.lang.String' to required type 'java.util.Date'**

意思就是前端表单传回来的String格式时间无法转换成Date，解决方法我一开始使用注解里的格式化，但对我无效，最后是使用了@InitBinder创建了一个专门初始化时间的方法来解决的
```java
@InitBinder
	public void initBinder(WebDataBinder binder, WebRequest request) {
		
		//转换日期
		DateFormat dateFormat=new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, true));// CustomDateEditor为自定义日期编辑器
	}
```
搞了好久就弄了个插入评论功能，还是不好做啊。明天准备先搞删除，然后实现点赞，最后看能不能把写评论换成ajax来实现，就不用多刷新页面了
## 第六天
### 第六天实现
- 增加评论删除功能
- 实现了ajax刷新评论点赞取消功能
- 修复了超过了8小时会掉链接出500错误的bug
***
修复8小时掉线，c3p0添加三个属性
```xml
	<property name="testConnectionOnCheckin" value="false"></property>
        <property name="testConnectionOnCheckout" value="true"></property>
        <property name="idleConnectionTestPeriod" value="4600"></property>
```
前端是真的费事。。。ajax如果用restful风格传参好像传不到controller里，还不知道原因，换成post提交表单可以。生成html代码很费劲。。拼接字符串之后用jquery获得div然后用html()函数修改。字符串双引号太多idea都识别不出来谁才是开头和结尾的双引号了。
## 第N天实现

- 修复了点赞的bug
- 为后端添加了数据验证（防止Postman直接提交表单绕过前端判断空）
- 优化Dao层代码，添加Good实体类
- 修改数据库设计，取消Common内部Good字段（实体类没变）
***
不知不觉过了一个学期了，后半个学期开发了好几个大项目。再看看半个学期前写的代码，不堪入目。首先JSON的返回格式没有规定好，有Success也有Error也有Message，以后我开发统一为三个字段，data，msg，code。Dao层写的也乱七八糟，返回值不该用boolean，应该用int判断。SSM还是有点麻烦，RestController返回的数据必须为Map类型，如果是fastjson还需要额外配置xml，除非导入jacksoncore包。

验证方面，我光给前端js加了验证，评论的前端甚至写的有问题，如果有人使用postman类似软件直接通过api提交表单，那么就会出错（同学经常这样做）233，添加了后端验证后就不会那么多事了。js和jquery当时果然不太熟，js操纵dom写的乱死了，竟然搞了15个变量来存新增的html代码串。

想想当初的zz操作，ajax不写error回调函数，还像个憨批一样到处说ajax难调试，控制台不输出，智障。

登录态检测不用Filter，反而在每个controller开头取session用if判断，简直是疯了。特别是这种前后端不分离的，后端跳转来跳转去自己都蒙了

Cookie保存登录状态不加密，这样别人直接修改用户名就自动登录别人的账号上了，安全性为零。

寒假再往上加东西的话，就考虑增加一个路径功能吧，也就是文件夹。这个慢慢构思下吧，感觉东西挺多的

更新记录应该写在一个文件里然后从后端读比较好，前端模板找个能支持md的就好了。

## 第NN天实现
由于腾讯云要到期了，不准备续费（因为薅了个免费阿里云的服务器）。所以准备搞个服务器迁移。由于老服务器是Windows server，而新服务器是Linux，所以改了下代码。

#### 判断系统
使用Java原生代码
```java
System.getProperty("os.name");//Linux返回Linux Windows10返回Windows 10
```
#### 数据库迁移
打了个.sql文件直接转，由于数据库里存的有文件路径，而Linux文件系统和Windows不太一样，所以把路径统一换了。Linux以/为根目录

#### 文件系统
Linux以/为根目录，文件分隔符为/，Windows以\分隔，可以从java中直接获取分隔符
```java
File.separator//使用separator属性直接获取当前系统的分隔符
```

#### 中文乱码
上传好的文件打了个压缩包，通过xshell传给了服务器，但是解压后发现中文文件名全部木大，都是乱码，百度了一下用convmv命令可以直接转。试了下centos7不带，所以还要先yum安装
```
yum install convmv  #使用yum安装convmv转码器

convmv -f GBK -t UTF-8 --notest -r /home/test #使用convmv命令转换编码为UTF-8,如果是目录则会全部转换
```

#### docker容器
在Linux我是用Docker来安装组件的，tomcat和mysql都是。使用容器数据卷连接一下tomcat的webapps目录，这样方便部署war包

跑了一下，发现登录不了。报错连接不上服务器，看了下代码。localhost没问题啊，都是服务器本机。仔细想了下，Docker容器是沙箱隔离的，一个容器的mysql和另一个容器的tomcat是访问不到的，所以把连接换成了服务器ip，成功登陆

上传，下载，都没问题，没想到这边改完一遍过了。但是下载之前的文件都不行，看数据库，路径没问题啊。想了想还是沙箱隔离的问题，就算运行的是tomcat，内部也有一个自己的centos，每次上传新文件都会到容器中的centos目录中，遂再搞一个容器数据卷，把本机的upload文件夹和tomcat的upload文件夹连接起来，再跑。没问题了。

## 面试准备

距离找工作面试的时间越来越近了，我准备把这个网盘也小改一下，加个文件系统的功能。

原本没有目录时，找文件是直接从数据库判断登录用户然后把所有文件都找到，但是加上目录就不行，因为不一定一个目录有所有的文件。

所以我使用了File类在磁盘里先找，找到所有的目录，标记为Directory类，文件也从磁盘找，找到文件后通过他的名字和登录用户名组合从数据库查找文件具体信息，再写入List，这样就不会影响到删除和下载，因为这两个是根据文件id起作用的。

今天加这个功能主要就卡在了前端，由于SSM很久没写了，jsp里的EL表达式出了很多问题，原来SpringBoot的Thymeleaf可以直接给js复制，通过双中括号的方式。所以我习惯就直接给js变量通过EL表达式赋值，结果因为我的值里有中文和英文冒号表示路径，卡了很久才解决。jsp还是推荐通过input type=hidden赋值，js用jquery选择器拿，这样可以解决编码问题。

返回上一级本来想的是记录一下lastPath，点击..就会回到lastPath变量的路径，可是这仔细一想叫后退，不叫返回上一级，如果连着点击两个..目录，你会发现你后退之后又退回去了，等于没动，因为实际上是后退操作。所以返回上一级应该从当前path的字符串扣一层路径才行，不能记录之前的值。

先这样吧，明天适配下Linux的文件系统，毕竟这个项目现在是在linux服务器上，而我做开发得先从本机windows系统来做，方便调试

## 远程tomcat调试

由于本机是Windows，所以对文件目录的操作只能以本机为准，可是服务器是Linux系统，两者的路径结构不同，写好了代码不好调试。上网一查发现可以远程tomcat调试。还是十分方便的

进入到tomcat的目录（docker 通过docker exec -it 容器id /bash/bin进入）

修改catalina.sh文件，添加以下字段

```sh
declare -x CATALINA_OPTS="-server -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8000"
```

最后的address改成你需要的端口，注意如果是阿里云需要添加相关安全组策略放开端口，非华为云需要打开防火墙对应端口。

如果你是Docker发现容器内无vim或vi命令，需要执行以下操作来添加vim

```bash
apt-get update
apt-get install vim
```

操作完成后即可使用vim操作文件。

放开tomcat调试端口后，在Intellij IDEA中添加远程debug模块

![远程debug idea选项](/img/onlineDisk/远程debug idea选项.jpg)

选择Remote

![idea remote](/img/onlineDisk/idea remote.jpg)

配置对应服务器tomcatIP和端口

![remote 配置](/img/onlineDisk/remote 配置.jpg)

添加完成后即可开始调试，记得打上断点~