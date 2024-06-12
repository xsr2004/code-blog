spring security1

做了三件事：认证，授权，保护

基于过滤器链，轻松集成

springboot底层默认

权限的分类

- 功能权限
- 访问权限
- 菜单权限



springboot集成thymeleaf中不能返回页面，只返回字符串

解决：@RestController换为Controller。因为前者是Controller + ResponseBody，和模板引擎没关系

https://blog.csdn.net/weixin_43731571/article/details/99970363



重要的类：

WebSecurityConfigurerAdapter：自定义Security策略

authenticationmanagerbuilder：自定义认证策略

WebSecurity模式



AbstractAnnotationConfigDispatcherServletInitializer

AbstractSecurityWebApplicationInitializer



**初始化，路由访问**





底层是aop

http访问对于anyRequest进行permitAll还是hasRole

基于内存数据的认证 - 基于jdbc的



**授权，认证**



加密方式：ctype，MD5



**注销，页面元素访问控制**



**csrf攻击**，logout在前端发的是一个get请求，由于csrf存在，可能存在用大量get请求去攻击服务器

springSecurity默认防护这个东西，所以在权限模块处理时不处理get

可以通过http.csrf().disable()去关闭csrf防护，这样get请求的logout也可以成功了



> 由于这里学习资料用的是thymeleaf，设计到模板引擎的一些语法如:sec和一些api等。
>
> 教程这里用的是前端去控制页面元素访问：即当登录后，判断当前登录用户的authentication，然后前端给加了这个display的隐藏。



**记住我**

当登录成功后，springSecurity通过往web-storage-cookies存一个{"remeberMe":value}

来实现用户下次关闭浏览器会话后，再次登录时直接默认登录。

http.remeberMe()



**指定登录页面和请求路由**

springSecurity默认提供一个/login路由和页面。当需要登录业务时，会访问/login页面，这个页面是默认的页面

> 各springSecurity版本的login.html都不太相同，越高版本样式越好看

我们可以自定义指定我们要的登录的login路由和登录表单。

```
//没有权限默认跳转到登陆页面,需要开启登陆的页面 默认会自动请求 /login, 可以用.loginPage指定跳转到登陆页请求的url
http.formLogin().loginPage("/toLogin").loginProcessingUrl("/login");
```

如上就是，登录表单为服务端 /toLogin端点，它会返回views/login页面（自定义登录页面）

```java
@RequestMapping("/toLogin")
public String toLogin(){
    return "views/login";
}
```

而访问时的请求url为/login。

> 同理，《记住我》要传的参数也可以用rememberMeParameter自定义
>
> 认证时的用户名和密码传参也是。



遇到的问题：

- @RestController和@Controller区别
  - 共同：表示Spring某个类是否可以接收HTTP请求
  - **二者区别： @RestController无法返回指定页面，而@Controller可以**；前者可以直接返回数据，后者需要@ResponseBody辅助。

- 依赖：模版引擎，springboot和security

- 降springboot版本以适应教程的security版本5.1.2
  - springboot 2.7.6 -> 2.6.4 用了parent标签的springboot更好平替
  - maven加载死慢
- pom不变蓝
  - idea中，pom.xml右键 -> 添加maven支持 -> 等待

https://portrait.gitee.com/MengZhongDeHaDeSen/springboot-security/blob/master/src/main/java/com/yi/config/SecurityConfig.java

https://mp.weixin.qq.com/s/FLdC-24_pP8l5D-i7kEwcg

https://www.bilibili.com/video/BV1KE411i7bC?p=4&vd_source=a312f003d7c3e57dfd813b31f9cd4a8e

https://www.cnblogs.com/east7/p/10462279.html