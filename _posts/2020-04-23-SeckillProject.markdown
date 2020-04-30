---
layout:     post
title:      "秒杀系统"
subtitle:   "基于SpringBoot实现"
date:       2020-4-23
author:     "TH"
header-img: ""
tags:
    - 项目
    - SpringBoot
    - 代码
    - 心得
---
# Java-SecKill

---

——*2020/4/23*   *Vue搞得有点难受*

---

## 引入Vue

因为想尽可能的前后端分离，所以引入了Vue。。

但是我的api都是RESTFul风格，所以还是有为了带参而存在的跳转Controller。

今天在使用Vue对界面进行绑定数据时发现很蛋疼的问题。Vue获取的json数据如果有两层，第一层可以直接取，但是第二层会报UndefinedType

解决就是在Vue对象的data属性中声明好数据结构，好像使用v-if也可以解决，不过我用了前者。

```json
goods:{
    good:{
        a:'',
        b:'',
    }
}
```

前端用得少就是惨啊，这bug调试了好久，我寻思着json也送到了啊，第二层也有怎么就未定义了。查了才知道有这情况。。

## Vo对象

数据库商品表分为两种，一种是秒杀商品表，里面定义了秒杀价和时间等数据，另一种是商品表，里面定义了商品的原价，id，描述等数据。

我在导入这个项目的前端文件发现，他是从一个实体类取得这两个表中的数据。我很懵，最开始我选择在秒杀商品实体类中在定义了一个普通商品的实体对象，然后用mybatis中map的association属性去另查，这样就可以查出两个表的数据了。首先前端就出了上面的双层json查不到属性问题。解决了之后我发现这不是一个长久之计。订单页面也要显示商品的数据。思考了一会我打算使用json数组，一个data对象中分别存商品的json和订单的json。刚准备写，我意识到这样子后端Controller封装json和前端取都有点麻烦。。最后我看了作者本人怎么处理的，发现他Controller最后发送的不是商品实体类也不是秒杀商品实体类，而是从vo包中拿出来的商品vo类。百度了一下才发现有这个概念。

> 一、**PO**
>
> persistant object 持久对象,可以看成是与数据库中的表相映射的java对象。最简单的PO就是对应数据库中某个表中的一条记录，多个记录可以用PO的集合。PO中应该不包含任何对数据库的操作。生命周期和数据库密切相关．在向数据库插入记录时创建该实体，删除或关闭数据库时该实体随之消亡．很多优秀的开源框架都实现了将数据库中的PO通过ORM用POJO来实际操作，如Hibernate,JDO等
>
> 二、**VO**
>
> value object值对象。通常用于业务层之间的数据传递，和PO一样也是仅仅包含数据而已。但应是抽象出的业务对象,可以和表对应,也可以不,这根据业务的需要.个人觉得同DTO(数据传输对象),在web上传递。

简单来说PO对象要和数据库一一对应，而VO就不一定对应了，可以根据需求添加自己需要的属性。所以就完美解决了上面的需求，建立一个新的商品VO类，把秒杀实体信息和商品实体信息揉到一块成这个对象再发给前端。这样前端也好从中取值。

---

——2020/4/27   axios怎么说呢，emmm...

---



## AXIOS和$.ajax()

Vue官方推荐axios来发送ajax信息，举出几个跟jquery的ajax不一样的地方吧。

1. axios指定请求方式的参数为method,jquery为type

   ```javascript
   $.ajax({
       type:'post'
   })
   ---
   axios({
       method:'get'
   })
   ```

   

2. axios传get请求参数为params，post请求为data，jquery统一为data(get发的是json字符串，post发序列化后结果)

   ```javascript
   axios({
       url:'xxx',
   	method:'post',
   	data:{
   		xxx:xxx,
   	}
   	method:'get',
   	params:{
   		xxx:xxx,
   	}
   })
   ---
   $.ajax({
       url:'xxx',
   	type:'get',
   	data:{
   		"xxx":"xxx",
   	}
   })
   ```

   

3. **axios在发post请求时出现的问题**

   axios在发送post时，是把data属性中的值转为json字符串在request的body体里发送，后端不加注解是无法直接封装的，所以需要加上@RequestBody注解，jquery的ajax默认传送json对象，后端不用加@RequestBody也可以封装，但如果想要使用@RequestBody注解，则需要指定contentType为application/json以及将json对象转换为字符串后即可。

   ```javascript
   $.ajax({
   	url : "/api/updateFeedback",
   	async : false,
   	type : "POST",
   	contentType : 'application/json',
   	dataType : 'json',
   	data :JSON.stringify(data),
   	success : function(data) {
   		lert("111");
   	}
   });
   ```

   

4. 回调函数

   axios在指定success回调函数时，可以直接使用then()方法，在指定error回调函数时，可以直接使用catch()方法，顺便说下，配合=>匿名函数更香哦。

   ```javascript
   axios({
       
   }).then(response=>{
      if(response.data.xxx){ //后端返回的数据会自动封装在response.data中，response里还有status这种属性
          xxx
      }   
   }).catch(response=>{
       console.log(response);
   });
   ```

---

——2020/4/28    第一次多并发实验

---

## 第一次多并发测试

现在的项目架构是最简单的，没有任何中间件或者负载均衡。来测试这种情况的高并发有没有bug或者看看极限吞吐量

![](/img/seckill/问题01-最初项目架构.png)

### 测试工具

使用Apache的jmeter来测试，直接拉满线程10000试试

结果如下

![](/img/seckill/问题1-03.jpg)

![](/img/seckill/问题1-02.jpg)

10000个测试线程有15.37%出错了，看了一下错误信息，没有返回值，仅仅抛了个异常，*Connection refused*。稍微查了一下，发现挺多问题都有这个Exception，隐隐感觉还是tomcat的配置问题，遂把tomcat最大连接数和线程数加大试试。

![](/img/seckill/问题1解决方法.jpg)

再试一次10000线程并发，没问题了

![](/img/seckill/问题1解决.jpg)

第一个问题就这么解决了，还是很高兴的。不过，吞吐量128。。emmm，而且出现了少卖或者超卖问题

![](/img/seckill/问题1-01.jpg)

没事，先加上Redis再搞token同步慢慢弄

加上缓存之后，超卖问题好像解决了，但是吞吐量变得更慢是什么鬼emmm，明天在研究吧。

## 第二次并发测试

---

——2020/4/29    第二次并发测试

---

昨天加上缓存并没有解决超卖问题，超卖问题实际上是因为下订单和减库存的顺序错了。虽然减库存的动作限制了库存数大于0才可以减，但这只保证了库存不会减到负数，并不能解决超卖问题。实际上解决的方法非常简单，**就是把他两个交换位置。先减库存，再下订单，这样不仅可以减少进入mysql的数据量，还能解决超卖问题。**

我今天优化了代码之后，保证没有多余的数据去写进mysql后，测试吞吐量是下图

![吞吐量低优化后纯走mysql](/img/seckill/吞吐量低优化后纯走mysql.jpg)

157.9，比昨天10几好多了，毕竟没用的数据不要进mysql才是正确的思路。

### 加缓存提速

昨天加了缓存，是将商品信息序列化为json存入redis string中。每次查询redis的商品信息再判断中间的库存数。

这个方法显然是不行的，商品库存作为一个超级热点数据，每秒都要修改几十次，那么存储成json的商品信息就要重新添加几十次，而且存储json相当于把其他属性再存一遍。

所以我决定修改下存储的结构，本来准备改成hash的，但是想着修改的数据就库存一条，其他的都可以在redis里长期存储不会变化。所以我把库存单独拉出来存到redis的string里，key是商品id，value就是库存值。

改完之后，由于库存是int，我想着SpringBoot Cache的序列化是可以处理的，没想到反序列化失败，redis里面取回来的String类型不能转换为int类型。

![SpringBootcache不会反序列化int](E/img/seckill/SpringBootcache不会反序列化int.jpg)

遂继续写自定义序列化方法的CacheManager，用jackson序列化Object即可

![自定义object序列化](/img/seckill/自定义object序列化.jpg)

### 自定义key前缀

SpringBoot Cache在你使用缓存注解时，如果没有指定自己的key生成器，SpringBoot会使用自动注入的生成器，cacheName和key之间会有两个冒号，

SpringBoot自动注入的简单生成器

![](/img/seckill/SpringBoot自动注入的simpleKey生成器.jpg)

simple实现

![simpleKey生成器](/img/seckill/simpleKey生成器.jpg)

所以只要自己写一个自定义的CacheKeyPrefix再通过RedisCacheConfiguration配置进去即可

![](/img/seckill/自定义key生成器.jpg)

只配一个冒号，之后在每个RedisCacheConfiguration配置中载入这个key生成器

![自定义key生成器配置](/img/seckill/自定义key生成器配置.jpg)

### ~~redis多线程并发~~

在添加上缓存之后再进行测试，发现会出现多线程并发问题。执行库存减一操作后发现日志吐出来的数据都是一样的。

![问题2redis高并发线程不安全](/img/seckill/问题2redis高并发线程不安全.jpg)

简单捋一下

在下单操作开始时，会从redis中读出库存数，如果存在即减一，不存在就查数据库并把数据回写到redis中。

redis库存减一后再把数据库的库存减一，如果减一失败，那么可能是数据库炸了，所以再将redis的库存加一。

结果再进行测试时，最后执行完线程组后，redis里的库存竟然还有数。

~~其实就是多线程的并发，在取出库存发现有数，但是在数据库减一之前就被其他的线程扣完了，所以数据库又失败了，这时候redis会再加一。造成最后执行结束还是个正数~~

这个问题我已经想好怎么解决了，使用Redis的List搞个安全token即可，还可以减少redis和mysql的写量，明天继续~

## 第三次并发测试

---

——2020/4/30    redis是个好东西

---

昨天最后发现日志输出的值都是一样的以为redis有并发问题，其实是没有的。SpringBoot2.0采用lettuce对redis连接，lettuce底层由netty实现，可以让一个链接多个线程共享。不会发生线程安全问题。那昨天日志打出来的一样的数字是怎么回事呢？

在高并发情况从redis取值是不可信的，库存为100的情况下，一下进来了1000个请求想要减库存。这1000个线程从数据库中取值，发现库存是100，然后就把redis的库存减到负数了。

所以我们要换下思路，尽量让少的请求来到DB，最好是有多少库存，最后就多少个请求进DB，其他的请求都在上游拦截。有两种思路可以实现

### 1.使用redis的nx来加锁

在decrement的操作进行时，先对这个库存加锁，set一个nx的值，这样减到0时停止秒杀就不会减到负数了

### 2.令牌桶原理

在redis的List中设置库存个uuid，每次有人想要进行下单操作时，先从该list中取出一个uuid，如果有uuid才可以进行下单和减库存操作。库存减到零时压根就不进入下单服务了。

我使用的是后者，我们可以在秒杀活动开始前进行预热，将uuid的set和list以及库存数存入redis。

![设置令牌桶](/img/seckill/增加redisList后性能.jpg设置令牌桶.jpg)

我们需要一个set来存某个商品的所有的uuid，这样才能保证不能让其他人随机生成一个uuid来冒充正常请求进入下单服务。

由于set的内部实现是hash，只占用hash的key而不是value来保证唯一，在进行ismember查询时计算hash并判断是否存在，所以时间复杂度是O(1)，性能很高，不用担心拉胯。

优化后的下单服务如下，在这个令牌桶生效的情况下，如果redis这边保持高可用不会停机，那么mysql可以完全舍弃（实现了redis数据库不会产生超卖少卖问题）

![加入令牌桶后的下单服务](/img/seckill/增加redisList后性能.jpg加入令牌桶后的下单服务.jpg)

当然mysql是不能省的，我们可以在中间加上消息队列，减缓进入mysql的数据量，并且redis插入成功后就异步返回成功下单提高吞吐率。

在增加了redis令牌桶限流情况下，吞吐量从昨天150多到了500，提升还是很大的，明天准备加MQ来异步提升吞吐

![](/img/seckill/增加redisList后性能.jpg)

