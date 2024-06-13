# spring事务

## spring如何进行事务管理的



主要是俩点，编程式实现和注解式实现。



编程式实现是通过spring提供的`TransactionTemplate`和`TransactionManager`来手动管理事务

TransactionTemplate#execute方法去执行需要事务的业务代码

也可以用TransactionManager的api，TransactionManager提供了获取事务，提交事务，回滚事务的接口，只需在这其中添加业务代码即可

> [关于`TransactionTemplate`和`TransactionManager`的简单例子参考这里](https://javaguide.cn/system-design/framework/spring/spring-transaction.html#spring-%E6%94%AF%E6%8C%81%E4%B8%A4%E7%A7%8D%E6%96%B9%E5%BC%8F%E7%9A%84%E4%BA%8B%E5%8A%A1%E7%AE%A1%E7%90%86)



注解式实现是通过spring提供的@Transcational注解来实现的（最常用）

@Transcational是spring的transaction模块的一个注解，用于提供声明式的事务支持

它的实现原理可以简述为：

> - 在bean初始化后的钩子函数中，会遍历容器所有的切面advice，将为 有@Transcational所在的方法的类/接口，生成代理对象
> - 当调用被@Transcational修饰的方法时，会走TransactionInterceptor事务增强操作，在执行目标方法的前后  进行事务处理（如开启，提交，回滚）。

> [关于`@Transcational的简单例子参考这里]((https://javaguide.cn/system-design/framework/spring/spring-transaction.html#spring-%E6%94%AF%E6%8C%81%E4%B8%A4%E7%A7%8D%E6%96%B9%E5%BC%8F%E7%9A%84%E4%BA%8B%E5%8A%A1%E7%AE%A1%E7%90%86))

![image-20240612203113681](D:\picGo\images\image-20240612203113681.png)



## 事务传播行为 （实战+分析）

本篇文章通过一个电商业务中的例子，引出事务传播在业务中的解决方式。

> [若要看知识点可以看](https://javaguide.cn/system-design/framework/spring/spring-knowledge-and-questions-summary.html#spring-%E4%BA%8B%E5%8A%A1%E4%B8%AD%E5%93%AA%E5%87%A0%E7%A7%8D%E4%BA%8B%E5%8A%A1%E4%BC%A0%E6%92%AD%E8%A1%8C%E4%B8%BA)
>
> ![图片](D:\picGo\images\640.webp)

**事务传播行为是为了解决业务层方法之间互相调用的事务问题**

比如这样的例子

管理员操作订单货物出库，

![image-20240612203947638](D:\picGo\images\image-20240612203947638.png)

本来的出库流程如下：

![image-20240612204021650](D:\picGo\images\image-20240612204021650.png)

而在上面的《修改订单状态》，《保存物流信息》，《记录变更日志》，《发送通知》这些操作当然可以放一个事务里，

但这样做的话，就会导致**任何一步出错，整个事务都会回滚**

比如《发送通知》这个操作出错了，我们希望一会重试一下就好了，因为它不是关键。若放在一个事务里，长事务带来的connection释放不掉问题导致数据库连接池爆满问题不说，关于《修改订单状态》，《保存物流信息》这些重要的信息也没有正确地执行。得不偿失。

解决上述方法

- **1.发布订阅模式，抛出事件，用异步监听器的方式去处理**（最常用）
- **2.开多个事务，比如《修改订单状态》和《保存物流信息》放一个事务，《记录日志》和《发送通知》放一个事务。不要“一个老鼠害一锅汤”。**

第一种权且不谈，本文旨在事务传播。

而对于第二种解决方式，

我们将这些“不重要的，可以容忍的”操作归纳为《其他操作》，它是一个事务（标记为B）

整个出库操作事务标记为A

会有下面的伪代码执行

```java
/**
* 出库操作
* @params itemId 出库商品id
*/
@Transcational//出库 事务 A
public void outStock(Long itemId){
	//更新订单状态
	updateOrderStatus();
	//保存物流信息
	saveTransportInfo();
	//记录日志和发送通知
	othersHandler();//其他操作 事务 B
}
```

>  在这个例子中，**我们希望的是事务B的执行不要影响A，但事务A的执行要影响到B**
>
> 比如订单状态都没有正常更新，那就不要发送通知，直接中断回滚就好了。
>
> 这里只是一个说明事务传播行为的demo，非真实正常业务。

也就是

![image-20240612205709163](D:\picGo\images\image-20240612205709163.png)

那么如何去做到`事务B的执行不要影响A，但事务A的执行要影响到B`

这时就需要规定**事务之间怎么进行传播**。

> 关于事务传播行为知识点和测试用例：
>
> https://segmentfault.com/a/1190000013341344#item-4
>
> https://javaguide.cn/system-design/framework/spring/spring-knowledge-and-questions-summary.html#spring-%E4%BA%8B%E5%8A%A1%E4%B8%AD%E5%93%AA%E5%87%A0%E7%A7%8D%E4%BA%8B%E5%8A%A1%E4%BC%A0%E6%92%AD%E8%A1%8C%E4%B8%BA
>
> 图来自https://juejin.cn/post/6844903929554141197?searchId=202406121923581693CCC81782E373E7EB#heading-5



所以我们可以将上述伪代码的事务传播参数设计为

> 《其他操作》事务为`Propagation.NESTED`，表示这是外围事务的嵌套子事务，当外围事务回滚，子事务也回滚；而当子事务回滚，在外围事务中catch住子事务的expection就ok，它（其他操作）单独回滚而不影响外围主事务和其他子事务

```java
/**
* 出库操作
* @params itemId 出库商品id
*/
@Transcational//出库 事务 A
public void outStock(Long itemId){
	//更新订单状态
	updateOrderStatus();
	//保存物流信息
	saveTransportInfo();
	//记录日志和发送通知
    try{
        othersHandler();//其他操作 事务 B
    }catch(othersHandlerExpection e){
        //catch 到 其他操作事务的 问题
    }
	
}

@Transcational(propagation = Propagation.NESTED)
public void othersHandler(){
    //dosomething
}

```

> 当然这种情况一般都用aop或者发布订阅模式去解决。



## 事务属性详解

隔离级别，超时规则，传播行为啥的



## @transcational基本用法

**注解用法：用于类，方法(最普遍)，接口**

详细介绍：

- `transactionManager()`表示应用那个应用那个`TransactionManager`.常用的有如下的事务管理器

- `isolation()`表示隔离级别，和mysql一样（串行化，可重复读(默认)，读已提交，读未提交）
- `propagation()`表示事务的传播属性，默认PROPAGATION_REQUIRED
- `rollbackFor()`，`noRollbackFor`回滚条件

> 默认情况下，如果在事务中抛出了未检查异常（继承自 RuntimeException 的异常）或者 Error，则 Spring 将回滚事务；除此之外，Spring 不会回滚事务。你如果想要在特定的异常回滚可以考虑`rollbackFor()`等属性

工作原理：

## @transcational原理实现(源码分析)



主要就是俩方面

**1.获取 该bean的切⾯⽅法**

在bean初始化后处理器中，先查出所有的切面(advisor)信息( this.advisorRetrievalHelper.findAdvisorBeans)，匹配当前bean中有@transcational的方法（threadlocal设置当前要处理代理的bean，canApply的判断方式是看能不能获取到TransactionAttribute），并把事务属性写入attributeCache。

这里先去看看advisor：

> `Advisor`是 spring-aop 原创的组件，**一个 Advisor = 一个 Advice Filter + 一个 Advice**。
>
> 在 spring-aop 中，主要有两种`Advisor`：`IntroductionAdvisor`和`PointcutAdvisor`。前者为`ClassFilter`+`Advice`，后者为`Pointcut`+`Advice`。https://www.cnblogs.com/ZhangZiSheng001/p/13745168.html

**2.创建aop代理对象：**

createProxy（创建proxyFactory，填充刚获取到的该bean的切面方法Advisor和TargetSource，getProxy），然后选择jdk或者cglib，生成该类的代理对象



![image.png](D:\picGo\images\c0e1839f5e5e46c8bfc4b697ee5683d4tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)



当执行主体函数时，

会被intercept||invoke拦截（看jdk/cglib），检验调用的方法修饰符(必须public)且有拦截器，调用TransactionInterceptor的invokeWithinTransaction方法

在这个方法中创建事务（获取事务，从cache中获取事务属性，设置，开启），然后调用methodProxy.invoke去执行业务逻辑，并对出错进行回滚处理，最后清空transactionInfo值，若执行成功则commit事务

下面是当调用@Transaction的方法时的处理：

![img](D:\picGo\images\webp-17182683616644-17182683714806-17182683933408-171826846195810-171826876942412.webp)

![image.png](D:\picGo\images\3f82db0be6a04aaeb9e8ba2323961834tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

总结：

`@Transactional` 注解，是使用 AOP 实现的，本质就是在目标方法执行前后进行拦截。在目标方法执行前加入或创建一个事务，在执行方法执行后，根据实际情况选择提交或是回滚事务。



## @transcational使用时的注意点

- `@Transactional` 只能应用到 public 方法才有效
  - 从动态代理本身来说：jdk基于接口实现代理，接口肯定是public的；cglib基于类继承实现代理，如果不是public子类都无法重写，更别提代理
  - 从源码角度：在匹配当前bean的@transcational方法的切面时，是通过看能否获取事务属性来判断的；而在获取事务属性时会先做校验，spring源码注释是// Don't allow non-public methods, as configured.。

- Spring AOP 存在自调用问题
  - 当方法被@transcational修饰后，其他类去调用这个方法时才会走代理对象的方法，通过invokeWithinTransaction方法来织入事务
  - 为什么？
    - 这是因为spring aop的工作原理是以类为代理对象的单位的，并非方法。
  - 解决方式：
    - 通过`AopContext.currentProxy()`获取当前类的代理对象，通过它来调用本类的@transcational方法
    - 自注入，不太符合面向对象的[设计原则]
    - 避免自调用的代码

- 不建议处理长事务
  - 若要用@transcational处理长事务则需要注意拆分和事务的生效
- 多事务时要注意传播行为的正确设置
  - 看是否结果被
- 要注意回滚条件的正确设置
  - 如果没正常回滚，很可能是catch异常被吞

## spring 事务没有正常生效的原因

- @transcational修饰了非public或者final，不管是jdk还是cglib代理模式，都不会生成代理对象。
- @transcational注解的方法所在的类 没有被spring管理
- 方法自调用
- 事务没有正常开启和设置（springboot默认开启）
- 异常被吞





## 长事务@Transactional

危害：

数据库连接池占满，容易死锁，回滚时间长....

解决：

编程式 ，手动管理事务的创建，关闭，commit，回滚逻辑等

声明式，对事务进行颗粒度更小的拆分，将大事务分为各小事务，并且要**保证事务@Transcational的生效**。





## 面试题

![image-20240612152114268](D:\picGo\images\image-20240612152114268.png)

![image-20240612152530140](D:\picGo\images\image-20240612152530140.png)



https://segmentfault.com/a/1190000013341344#item-4

https://javaguide.cn/system-design/framework/spring/spring-transaction.html#transactional-%E6%B3%A8%E8%A7%A3%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3

https://github.com/TmTse/transaction-test

https://juejin.cn/post/7208479235132244023#heading-0

https://juejin.cn/post/6844903929554141197?searchId=202406121923581693CCC81782E373E7EB

https://www.javadoop.com/