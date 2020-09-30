---
layout:     post
title:      "使用SpringCloudOauth2配合JWT实现微服务权限验证（密码型）"
subtitle:   "微服务"
date:       2020-09-30
author:     "TH"
tags:
    - 项目
    - SpringCloud
    - 代码
    - 心得
---
# 前言

OAuth 2.0指的是认证方式

专用名词 引用一下大佬的博客

> （1） **Third-party application**：第三方应用程序，本文中又称"客户端"（client），即上一节例子中的"云冲印"。
>
> （2）**HTTP service**：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的Google。
>
> （3）**Resource Owner**：资源所有者，本文中又称"用户"（user）。
>
> （4）**User Agent**：用户代理，本文中就是指浏览器。
>
> （5）**Authorization server**：认证服务器，即服务提供商专门用来处理认证的服务器。
>
> （6）**Resource server**：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

一般有以下几个方式来进行验证服务

> 客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。
>
> - 授权码模式（authorization code）
> - 简化模式（implicit）
> - 密码模式（resource owner password credentials）
> - 客户端模式（client credentials）

引用文章出处：http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

# 项目架构

![](/img/springcloudoauth2/1.jpg)

## Eureka-server

### 服务中心

服务中心配置yaml文件

```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
spring:
  application:
    name: eureka-server

```

启动类注解

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

## server-auth认证服务器

### 认证服务器配置 AuthorizationServerConfigurerAdapter

```java
@Configuration
@EnableAuthorizationServer //一定不要忘记这两个注解
public class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {

    @Autowired
    AuthenticationManager authenticationManager; //密码模式需要manager

    @Resource
    RedisTokenStore tokenStore; //存储token的位置

    @Bean
    public JwtAccessTokenConverter accessTokenConverter(){//AES验证方式，秘钥设置
        JwtAccessTokenConverter accessTokenConverter=new JwtAccessTokenConverter();
        accessTokenConverter.setSigningKey("hi");
        return accessTokenConverter;
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.allowFormAuthenticationForClients()//允许表单登录验证，不调用此项无法使用PostMan调试
                .tokenKeyAccess("permitAll()")
                .checkTokenAccess("permitAll()");
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory() //inmemory模式表示在内存中
                .withClient("user-client")//id
                .authorizedGrantTypes("password","refresh_token")//模式
                .scopes("all")//域
                .authorities("oauth2")
                .secret(new BCryptPasswordEncoder().encode("hi"))//秘钥
                .accessTokenValiditySeconds(60*60*30) //过期和刷新时间
                .refreshTokenValiditySeconds(60*60*30)
                .redirectUris("www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager)//密码模式需要配置
                 .tokenStore(tokenStore)
                 .userDetailsService(new MyUserService());

    }
}

```

### 用户验证过程服务类UserDetailsService

```java
@Service("my_user_service")
public class MyUserService implements UserDetailsService {

    @Resource
    PasswordEncoder myPasswordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        if(s.equals("user")) {
            String name = "user";
            String password = myPasswordEncoder.encode("hi_user");
            String role = "user";
            List<SimpleGrantedAuthority> list = new ArrayList<>();
            list.add(new SimpleGrantedAuthority(role));
            return new User(name, password, list);
        }else if(s.equals("admin")){
            String password=myPasswordEncoder.encode("hi_admin");
            List<SimpleGrantedAuthority> list=new ArrayList<>();
            list.add(new SimpleGrantedAuthority("admin"));
            return new User("admin",password,list);
        }else{
            return null;
        }
    }
}
```

通过username查询，正常应该走数据库，这里是demo就写死在代码中，只有user和admin两个用户

**注意Role和Authority是两种属性！！！！！**

### TokenStore和加密Encoder配置

```java
@Configuration
public class RedisTokenStoreConfig {

    @Resource
    private RedisConnectionFactory redisConnectionFactory;

    @Bean
    public TokenStore tokenStore(){
        return new RedisTokenStore(redisConnectionFactory);//使用Redis存储
    }

    @Bean
    public PasswordEncoder myPasswordEncoder(){
        return new BCryptPasswordEncoder();//Hash加密类，每次加密秘钥都不同，随机盐
    }

}
```

### WebSecurityConfigurerAdapter安全配置

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Resource(name = "my_user_service")
    private UserDetailsService userDetailsService;

    @Resource
    private PasswordEncoder myPasswordEncoder;

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider(){//自定义密码模式验证过程
        DaoAuthenticationProvider daoAuthenticationProvider=new DaoAuthenticationProvider();
        daoAuthenticationProvider.setUserDetailsService(userDetailsService);
        daoAuthenticationProvider.setPasswordEncoder(myPasswordEncoder);
        daoAuthenticationProvider.setHideUserNotFoundExceptions(false);
        return daoAuthenticationProvider;
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/**").permitAll();//认证服务器自身不需要权限

    }
}
```

### 启动类

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients//不要忘记这个，要不然会有IOC注入异常
public class ServerAuthApplication {

	public static void main(String[] args) {
		SpringApplication.run(ServerAuthApplication.class, args);
	}

}
```

yaml配置

```yaml
server:
  port: 8763

spring:
  application:
    name: server-auth
  redis:
    host: 47.98.182.40
    port: 6379
    password: root

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

## service-hi资源服务器

ResourceServerConfigurerAdapter资源服务器配置类

```java
@Configuration
@EnableResourceServer//不要忘记这两个注解
public class ResourceConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/userHi").hasAuthority("user")
                .antMatchers("/adminHi").hasAuthority("admin");
    }
}
```

### service

```java
//服务类
@Service
public class HiServiceImpl implements HiService {
    @Override
    public String adminHi(String name, Integer port) {
        return "hi i am admin "+name+" from port "+port;
    }

    @Override
    public String userHi(String name, Integer port) {
        return "hi i am user"+name+" from port "+port;
    }
}

```

### **资源服务器yaml配置 重点**

```yaml
server:
  port: 8762

spring:
  application:
    name: service-hi
  redis:
    host: 47.98.182.40
    port: 6379
    password: root

security:
  oauth2:
    client:
      client-id: user-client #客户端id
      client-secret: hi #客户端秘钥
      access-token-uri: http://localhost:8763/oauth/token #获取认证服务器token地址
    resource:
      id: user-client
      token-info-uri: http://localhost:8763/oauth/check_token #认证服务器检查token地址
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/


```

## gateway网关

### 请求拦截器RequestInterceptor

在feign传送过程中，是无法把Header中的token传过去的。需要配置请求拦截器手动将header带入

```java
/**
 * @author han.tian
 * @date 2020.9.30
 *  必须配置拦截器，使oauth2的token可以通过feign发送给别的服务
 */
@Component
public class JWTInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate requestTemplate) {
        RequestAttributes requestAttributes=RequestContextHolder.currentRequestAttributes();
        HttpServletRequest request = ((ServletRequestAttributes) requestAttributes).getRequest();
        String authorization = request.getHeader("Authorization");
        requestTemplate.header("Authorization", authorization);
    }
}
```

### 代理service

```java
@FeignClient("service-hi")//指定微服务名称
@Service
public interface HiService {

    @RequestMapping("/adminHi")
    String home(@RequestParam String name);

    @RequestMapping("/userHi")
    String userHome(@RequestParam String name);

}
```

### yaml

```yaml
server:
  port: 8764

spring:
  application:
    name: server-gateway

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

