---
layout:     post
title:      "百度贴吧签到springboot启动器+Nexus配置"
subtitle:   "springboot自动配置"
date:       2020-4-17
author:     "TH"
header-img: ""
tags:
    - 项目
    - SpringBoot
    - 代码
    - 心得
---
## 百度贴吧自动签到脚本

之前闲着蛋疼用Java搞了个百度贴吧自动签到的脚本，使用HTTPClient发送请求，JSoup处理HTML，FastJson处理json数据。

本来这个脚本是从properties文件中读取数据，别的项目导jar包来引入，总感觉挺麻烦的，想融到springboot里去，使用spring的定时任务和自动配置类来让这个项目更方便些。于是就打算写一个启动器（starter）。

## SpringBoot启动器

先来看一下项目的具体架构

![项目架构](\img\augosigned-starter-01.jpg)

**可以看到starter只是一个依赖控制的作用，他自己是个空项目，通过pom文件来引入子自动配置类，当然我们后面也可以添加其他的组件，然后使用starter的pom连接起来。其他人在使用的时候不用管内部有什么组件，只需要引入starter就可以一次性把所有的组件都引入进来。**

### xxxProperties类

properties类内部的属性就是我们要往application.yml中添加的配置选项。

通过@ConfigurationProperties注解实现导入配置文件

```java
@ConfigurationProperties(prefix = "cn.th.bdautosigned")
public class BdAutoSignedProperties {
    private String bduss;

    public String getBduss() {
        return bduss;
    }

    public void setBduss(String bduss) {
        this.bduss = bduss;
    }
}
```

上面我们把cn.th.bdautosigned添加到了注解的prefix属性中，这样我们只要在yml中添加这个值就可以自动导入到类中了

### xxxAutoConfiguration类

这就是我们最熟悉的自动配置类了，那么他是怎么实现自动配置的呢？

首先认识一下@EnableConfigurationProperties注解，这个注解帮助我们导入对应的Properties文件，说白了就是自动引入对应的类。

```java
@EnableConfigurationProperties(BdAutoSignedProperties.class)
```

这样写我们就可以把BdAutoSignedProperties这个类自动引入进来，可以直接用@Autowired或者@Resource注入了。

@ConditionOnxxxx

这是一套Condition的注解，有@ConditionOnMissingBean，@ConditionOnClass，总之这个注解就代表了一个判断，比如@ConditionOnMissingBean(BdAutoSignedAutoConfiguration.class)这个注解，如果写到我们的自动配置类上，那么这个自动配置类在被Spring加载时，他会检查容器中有没有BdAutoSignedAutoConfiguration类，如果存在，那么这个自动配置类就不会加载。

这其实就是SpringBoot的一个核心思想，如果用户自己配置了相关的配置类时，他会把我们开发者写的类失效，转而使用用户自己写的配置类。

另外，在编写配置类后，需要在classpath:META-INF/spring.factories文件中添加自己自动配置类的全限定类名

```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  cn.th.bdautosigned.starter.BdAutoSignedAutoConfiguration
```

这块实际上牵扯到了SpringBoot的启动原理，在SpringBoot启动时，他会把spring.factories文件中的所有全限定类名都加载进来，之后就根据自动配置类的@Condition注解来判断当前环境符不符合自动配置类的需求，再加载或者抛弃。

来看看我脚本的自动配置类

```java
@Configuration
@ConditionalOnMissingBean(BdAutoSignedAutoConfiguration.class)
@EnableConfigurationProperties(BdAutoSignedProperties.class)
public class BdAutoSignedAutoConfiguration {
    @Resource
    BdAutoSignedProperties properties;

    @Bean
    @ConditionalOnMissingBean(AutoSignedService.class)
    public AutoSignedService service(){
        return new AutoSignedService(properties);
    }

}
```

导入了自动配置类后，他会帮我们new AutoSignedService类，实现了Spring的控制反转。

## Maven私服（Nexus）

写好了SpringBoot启动器后，我可以通过Maven的Install功能安装到本地仓库了，说实话，在主项目中配一下pom就把所有的东西都拉进来并且自动配置好的感觉，实在爽到。但是这还是有不足，比如换个电脑就不行了，就必须从git上把所有的项目拉下来再install后才能使用。这不符合我的目的。所以我就开始搭建Maven私服了。

我是选择在服务器上用Docker来一键搭好（Docker大法好），搭好后记得从硬盘本地查看一下第一次搭建的随机密码。

### nexus的仓库类型

![](\img\autosigned-stater-02.jpg)

#### 1.proxy代理仓库

代理仓库顾名思义，就是从第三方代理仓库下载jar包的意思，在我们pom文件写完要导入一个jar包时，他会先从本地仓库找对应的jar包，如果没找到，就会从远程仓库中寻找，如果我们自己的服务器仓库也没有这个jar包，那么代理仓库就会从自己代理地址里的仓库查找对应的jar包，找到后就会下载到自己的服务器，节省下一次的带宽消耗。

#### 2.hosted仓库

就是宿主仓库的意思，在这个仓库中我们可以上传自己的jar包，正好对应我的需求。注意我们如果要上传jar包的话，它是分为两个版本的，一个是**SNAPSHOT测试版**，另一个是**RELEASE正式版**在我们创建maven工程时他都会提醒你设置版本号，以及版本信息（正式版还是测试版），区别就在这点，再上传jar包是他会判断我们的版本信息，RELEASE版本的jar包会到专门的releases仓库，SNAPSHOT版本就会传到snapshots仓库，如果是SNAPSHOT版本但不配置SNAPSHOT仓库的话，Deploy时会出错哦~

![](\img\augosigned-starter-03.jpg)

#### 3.group组仓库

![](\img\augosigned-starter-04.jpg)

就是把多个仓库揉成一个仓库组，在配置下载仓库时比较方便，组仓库会有一个下载顺序，比如上图，他会先从releases库中寻找jar包，找不到再从snapshots库中找jar包，最后找不到通过代理仓库到第三方仓库继续找。

### Settings.xml配置

maven中配置文件经常遇到的有两个，一个是pom.xml，另一个是Settings.xml，这两个文件分别对应局部和全局，pom配置的就是单独某个项目的配置，settings配置的是所有项目通用的配置。

我们配置私服后，需要对应的账号密码才可以访问仓库，而这个数据配置在settings文件中，因为pom文件要上传上去，所以配置在settings里安全

![](\img\augosigned-starter-05.jpg)

id指定对应服务器的特别标示，在pom里要制定上传仓库的话，id要和settings中的一致，这样才知道对应的服务器账号密码是什么

#### 下载

如果我们要从第三方的服务器下载，需要在settings文件中配置对应的mirror配置

![](\img\augosigned-starter-06.jpg)

mirrorOf属性制定的是我们访问哪个远程仓库的时候需要走这个代理，我设置*代表所有远程仓库都不走，都走这个代理服务器。**注意id要和server中的一致**

### pom.xml配置

#### 上传

要上传jar包到服务器（deploy）的话，我们需要在pom中指定上传的仓库地址

![](\img\augosigned-starter-07.jpg)

**注意id也要和settings对应的id一致，这样才能拿到对应服务器的验证密码。**

之后就没问题了，deploy上传到仓库后，我们就可以在别的电脑上配置我们的nexus再从上面拉项目了。一般公司都是有内部的maven服务器，这样来共享项目jar包，并且可以保密不透露给外界。