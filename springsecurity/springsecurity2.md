## cookie，session，token，jwt



https://www.cnblogs.com/tomyyyyy/p/15134420.html#jwt-tool

https://www.bilibili.com/video/BV1ob4y1Y7Ep/?spm_id_from=333.337.search-card.all.click&vd_source=a312f003d7c3e57dfd813b31f9cd4a8e

https://jwt.io/

https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html

https://developer.aliyun.com/article/995894

![image-20240515150611945](D:\picGo\images\image-20240515150611945.png)

![image-20240515151818788](D:\picGo\images\image-20240515151818788.png)

![image-20240515154309718](D:\picGo\images\image-20240515154309718.png)

基于fastjson的redis序列化器

工具类：jwt，redisCache，webUtils

实体类：user

响应封装



配置mysql，redis，mybatis，测试crud

userDetails，配置loginUser，securityConfig

encoder

登录接口：

放行login接口，其他路径要authentication

authentication的实现类UsernamePasswordAuthenticationToken，让manager去验证，这时会调用UserDetailsService的loadByUsername去验证，通过则返回一个附带权限的authentication

如果authentication没问题，则principal会有一个loginUser，你需要根据loginUser的username或者其他的去create一个jwt。并封装到结果中，

最后把这个jwt存进redis，key为login:+userId，value为这个token。

将结果封装返回。

![image-20240515212357462](D:\picGo\images\image-20240515212357462.png)

OncePerRequestFilter的使用

![image-20240515214316962](D:\picGo\images\image-20240515214316962.png)

关于permitAll和anoymous

关于



退出登录

- 取出securityContextHolder中的authentication，取出loginUser的userId
- 删除redis的login的key



授权

![image-20240516115817103](D:\picGo\images\image-20240516115817103.png)

![image-20240516120625005](D:\picGo\images\image-20240516120625005.png)

![image-20240516120633391](D:\picGo\images\image-20240516120633391.png)

![image-20240516120713567](D:\picGo\images\image-20240516120713567.png)

SimpleGrantedAuthority没必要序列化

![image-20240516120134792](D:\picGo\images\image-20240516120134792.png)

权限信息封装

- loginUser封装permmission，封装权限信息到GrantedAuthen中
- 登录时，从数据库查出后，存到loginuser的getAuthentic方法中，并把权限信息封装到filterToken中
- 接口处加一个@PreAuthorize
- 记得开权限相关配置



springboot和springSecurity版本对应

SecurityContextHolder原理



SecurityExpressionRoot规定了一些权限鉴定方法，如hasAuth，hasRole，用spring el表达式处理



rabc，用户，角色，权限表，用户角色关联，角色权限关联



自定义异常处理器

 我们还希望在认证失败或者是授权失败的情况下也能和我们的接口一样返回相同结构的json，这样可以让前端能对响应进行统一的处理。要实现这个功能我们需要知道SpringSecurity的异常处理机制。

 在SpringSecurity中，如果我们在认证或者授权的过程中出现了异常会被ExceptionTranslationFilter捕获到。在ExceptionTranslationFilter中会去判断是认证失败还是授权失败出现的异常。

 如果是认证过程中出现的异常会被封装成AuthenticationException然后调用**AuthenticationEntryPoint**对象的方法去进行异常处理。

 如果是授权过程中出现的异常会被封装成AccessDeniedException然后调用**AccessDeniedHandler**对象的方法去进行异常处理。

 所以如果我们需要自定义异常处理，我们只需要自定义AuthenticationEntryPoint和AccessDeniedHandler然后配置给SpringSecurity即可。



允许跨域

```
   http.cors();
```

自定义权限校验方法hasAuthority



认证成功/失败处理器

 实际上在UsernamePasswordAuthenticationFilter进行登录认证的时候，如果登录成功了是会调用AuthenticationSuccessHandler的方法进行认证成功后的处理的。AuthenticationSuccessHandler就是登录成功处理器。AuthenticationFailureHandler就是登录失败处理器。

```
http.formLogin()
//                配置认证成功处理器
                .successHandler(successHandler)
//                配置认证失败处理器
                .failureHandler(failureHandler);
```

问题：

下面代码发生栈溢出

```
UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken = new UsernamePasswordAuthenticationToken(user.getUserName(), user.getPassword());
Authentication authenticate = authenticationManager.authenticate(usernamePasswordAuthenticationToken);
```

**userDetails接口提供的isEnabled默认是false，账户不可用，去自定义时一定要改为true，否则你去访问就是没任何反馈，也没有报错。**

![image-20240515185910153](D:\picGo\images\image-20240515185910153.png)

![image-20240515185902535](D:\picGo\images\image-20240515185902535.png)

> 至于，栈溢出的原因，不知道，估计是成circle循环调用了.....

项目jdk检查

**nested exception is java.lang.NoClassDefFoundError: javax/xml/bind/DatatypeConverter**

> jjwt默认使用的是JAXB API是java EE 的API去compact()，而jdk8以上就没有了。需要手动添加。
>
> 但我的pom.xml，settings，项目结构里的模块都是jdk8，结果项目是21。
>
> 我还是通过Springboot启动的时候发现的。

然后在全局设置那改项目jdk就ok了

![image-20240515185819572](D:\picGo\images\image-20240515185819572.png)



解决com.alibaba.fastjson.JSONException：autoType is not support问题，Redis FastJson

![image-20240516115037186](D:\picGo\images\image-20240516115037186.png)

全局捕获不到 todo



https://blog.csdn.net/qq_32510597/article/details/106021933

https://www.bilibili.com/video/BV1mm4y1X7Hc?p=39&vd_source=a312f003d7c3e57dfd813b31f9cd4a8e



