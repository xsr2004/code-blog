springSecrutiy文档

源码解析，设计思想。



待写博客

1.cookie，token，session，csrf √

2.springSecurity如何实现认证流程

RABC权限设计

3.springSecurity思想

4.重要源码和图解



https://www.cnblogs.com/zlaoyao/p/16588950.html

AuthenticationProvider 

AuthenticationDetailsSource和AuthenticationDetails

自定义 LogoutHandler

LDAPhttps://cloud.tencent.com/developer/article/2055095

https://javaforall.cn/

## security工作流程

![image-20240518135602607](D:\picGo\images\image-20240518135602607.png)

![spring security 过滤器链嵌入原理_firewalledrequest-CSDN博客](D:\picGo\images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NDA0MjU4,size_16,color_FFFFFF,t_70.jpeg)



每个请求到达服务端的时候，首先从session中找出SecurityContext ，为了本次请求之后都能够使用，设置到SecurityContextHolder 中。

当请求离开的时候，SecurityContextHolder 会被清空，且SecurityContext 会被放回session中，方便下一个请求来获取

## RABC权限设计

## Session Fixation攻击

## Remember-Me认证和存在的问题/优化

## CSRF是什么，security如何防止

## 遇到的最大挑战是什么

Spring Security 登录成功后总是获取不到登录用户信息？

也许是对登录路径设置了web.ignore.antMachers("/login")。

而web.ignore是不走spring security的。

而对于处理登录信息的过滤器**`SecurityContextPersistenceFilter`** ，

> 每一个请求到达服务端的时候，首先从 session 中找出来 SecurityContext ，
>
> 然后设置到 SecurityContextHolder 中去，方便后续使用，
>
> 当这个请求离开的时候，SecurityContextHolder 会被清空，SecurityContext 会被放回 session 中，方便下一个请求来的时候获取。

如果用web.ignore就不走这个SecurityContextPersistenceFilter了，就算上次登录了，也是假登录，session没数据的。也不会将session的SecurityContext设置回SecurityContextHolder 



https://juejin.cn/post/6844904111393996813

## security单元测试

https://blog.csdn.net/qq_41076797/article/details/86506732

https://blog.csdn.net/z69183787/category_2175245.html

https://blog.csdn.net/qq_35067322/article/details/117433383

CurrentSecurityContext

https://blog.csdn.net/Aqting/article/details/125857193

SpringSecurity中的Authentication信息与登录流程https://www.cnblogs.com/summerday152/p/13636285.html

## security如何保存用户信息，如何获取用户信息（一）（含部分securityHolder源码分析）

先贴一段适应性高的代码

```java
@Override
public UmsMember getCurrentMember() {
    SecurityContext ctx = SecurityContextHolder.getContext();
    Authentication auth = ctx.getAuthentication();
    MemberDetails memberDetails = (MemberDetails) auth.getPrincipal();
    return memberDetails.getUmsMember();
}
```

在AbstractAuthenticationToken之前，会经过SecurityContextPersistenceFilter 。

SecurityContextPersistenceFilter 先从HttpSession中读取SecurityContext出来，存入SecurityContextHolder以备后续使用，当请求离开SecurityContextPersistenceFilter的时候，获取最新的SecurityContext并存入HttpSession中，同时清空SecurityContextHolder中的登录信息

但对于我们的前后端分离的情况，一般都用jwt，

**而spring security默认用httpSession来保存认证信息**。

可以通过WebSecurityConfigurerAdapter#configure的httpSecurity来设置《无会话状态》

```java
@Configuration
public class MySecurityConfiguration extends WebSecurityConfigurerAdapter {
    // code...
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .sessionManagement()
            	//设置无状态，所有的值如下所示。
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
    }
    // code...
}
```

这里要提一下SecurityContextPersistenceFilter的源码，主要关注这一行SecurityContext contextBeforeChainExecution = this.repo.loadContext(holder);

```java
SecurityContext contextBeforeChainExecution = this.repo.loadContext(holder);
try {
    SecurityContextHolder.setContext(contextBeforeChainExecution);
    if (contextBeforeChainExecution.getAuthentication() == null) {
        logger.debug("Set SecurityContextHolder to empty SecurityContext");
    }
    else {
        if (this.logger.isDebugEnabled()) {
            this.logger
                .debug(LogMessage.format("Set SecurityContextHolder to %s", contextBeforeChainExecution));
        }
    }
    chain.doFilter(holder.getRequest(), holder.getResponse());
}
```

这个this.repo是一个

```
private SecurityContextRepository repo;
```

SecurityContextRepository是操作SecurityContext的mapper，security提供了俩种对SecurityContext的操作实现类（如图）

![image-20240519142633869](D:\picGo\images\image-20240519142633869.png)

若开启session(默认)，那么每次

- 先从HttpSession中读取SecurityContext出来，存入SecurityContextHolder以备后续使用，当请求离开SecurityContextPersistenceFilter的时候，获取最新的SecurityContext并存入HttpSession中，同时清空SecurityContextHolder中的登录信息

不开启session，那么每次

- 通过SecurityContextHolderStrategy的策略选择，去创建一个空context，存入holder



## security如何保存用户信息（二）（主要是securityHolder策略模式分析）



**先大概看一下策略模式：**

策略模式包含以下几个核心角色：

- **环境（Context）**：维护一个对策略对象的引用，负责将客户端请求委派给具体的策略对象执行。环境类可以通过依赖注入、简单工厂等方式来获取具体策略对象。
- 抽象策略（Abstract Strategy）：定义了策略对象的公共接口或抽象类，规定了具体策略类必须实现的方法。
- **具体策略**（Concrete Strategy）：实现了抽象策略定义的接口或抽象类，包含了具体的算法实现。

![image-20240519152806042](D:\picGo\images\image-20240519152806042.png)

太抽象了。和dy一样。



**直接看一下ContextHolder里的策略模式**

![image-20240519151704703](D:\picGo\images\image-20240519151704703.png)

**分析：ContextHolder里有一个strategy的引用，这个strategy有很多实现类，对应不同的情况策略**

**ContextHolder会在初始化时，将用户选择的strategy实现注入，让strategy去执行对SecurityContext的操作。**

它提供了三种策略来保存springSecurity：

- MODE_THREADLOCAL ：将SecurityContext放在ThreadLocal中，开启子线程，子线程获取不到用户数据。

- MODE_INHERITABLETHREADLOCAL：多线程环境，子线程也能获取到用户数据。

- MODE_GLOBAL：数据保存到一个静态变量中，web开发中很少使用

**具体看一下代码：**

SecurityContextHolder在初始化时，会读取`spring.security,strategy`参数并对strategy进行初始化。![image-20240519151932232](D:\picGo\images\image-20240519151932232.png)

![image-20240519151857558](D:\picGo\images\image-20240519151857558.png)

> 一般默认是用ThrealLocal存context 这种策略的



然后在执行对context的行为时，根据所选策略类来执行。

```java
public static void clearContext() {
    strategy.clearContext();
}
public static SecurityContext getContext() {
    return strategy.getContext();
}
```

比如ThreadLocalSecurityContextHolderStrategy：

```java
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {
    private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal();

```

- 存储载体是ThreadLocal 针对SecurityContext的操作都是在ThreadLocal中进行操作。SecurityContext只是个接口，只有一个实现类是SecurityContextImpl

- **InheritableThreadLocalSecurityContextHolderStrategy** ：存储载体为InheritableThreadLocal ，InheritableThreadLocal继承ThreadLocal，多了一个特性，就是在子线程创建的时间，会自动将父线程的数据复制到子线程中。实现了子线程中能够获取登录数据的功能。





https://juejin.cn/post/6980101429604122631?searchId=20240519140930FAB353BCCABDF9BD2FA3

## oath2

![image-20240519145354964](D:\picGo\images\image-20240519145354964.png)