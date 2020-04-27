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

   