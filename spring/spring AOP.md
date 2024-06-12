# AOP是什么

AOP，面向切面编程，是一种编程理念。

AOP 中最小的单元是“aspect”，一个aspect包含很多类型和对象。

# 为什么要引入AOP

因为java的oop面向对象编程有下面问题

- 侵入性扩展：扩展需要影响源码
- 静态化语言：类结构一旦确定，不修改

通过 AOP 我们可以**把一些非业务逻辑的代码（比如安全检查、监控等代码）从业务中抽取出来**，以非入侵的方式与原方法进行协同。这样可以使得原方法更专注于业务逻辑，代码接口会更加清晰，便于维护。

# 简述 AOP 的使用场景

日志场景：诊断上下文.......

统计场景：耗时，执行/异常次数，数据抽样......

性能场景：缓存，超时控制

# AOP的几个概念

**aspect切面：**切面类，具体到代码就是一个aspect类，它里面定义了对于pointCut的joinpoint去执行怎样的advice。

**joinPoint连接点：**切面和被代理类某方法的连接点，程序执行时的一个点，spring AOP仅支持方法级别

**pointCut切点：**符合pointCut条件的joinPoint才能被advice执行。

**advice通知：**对于joinPoint的横切逻辑，有@before，@Aroud等，不想和业务代码耦合的代码可以放在这里

**Weaving植入**：将aspect和joinpoint对应的方法关联，让advice得以执行。

**AOP proxy**：通过实现AOP对应协议生成的 代理对象

https://www.cnblogs.com/lifullmoon/p/14654795.html#%E4%BB%80%E4%B9%88%E6%98%AF-aop

https://segmentfault.com/a/1190000007469982#item-1-2

# 静态代理和动态代理

## 静态代理

![image-20240521110534874](D:\picGo\images\image-20240521110534874.png)

> 举个例子：乘客每次要去火车站去买票，但火车站流量太大，
>
> 为了缓解，可以让其他代售点（去哪儿，携程）去代卖，
>
> 乘客就从代售点卖即可

**1.首先定义一个公共接口，表示火车站售票服务。**

```java
public interface TicketService {
    void buyTicket();
}
```

**2. 实现被代理类**

实现接口的实际火车站售票类，包含实际的售票逻辑。

```java
public class RealTicketService implements TicketService {
    @Override
    public void buyTicket() {
        System.out.println("Ticket has been successfully purchased from the station.");
    }
}
```

**3. 实现代理类**

实现接口的在线售票代理类，包含对实际火车站售票服务的引用，并在方法调用前后执行额外的操作。

```java
//去哪儿 互联网代售点
public class TicketServiceProxyGoWhere implements TicketService {
    private RealTicketService realTicketService;

    public TicketServiceProxy(RealTicketService realTicketService) {
        this.realTicketService = realTicketService;
    }

    @Override
    public void buyTicket() {
        System.out.println("这里是 去哪儿 12306代售点");
        realTicketService.buyTicket();
        System.out.println("欢迎使用 去哪儿 代售服务...");
    }
}
```

## 动态代理

1. **通过实现接口的方式 -> JDK动态代理**
2. **通过继承类的方式 -> CGLIB动态代理**

### jdk

实战：

我们实现InvocationHandler类的invoke方法

通过`Proxy.newProxyInstance`方法，传入代理类.classloader和interfaces和handler，生成 一个 代理类，cast为被代理类实现的接口

```java
//抽象角色：租房
public interface Rent {
   public void rent();
}
```

```java
//真实角色: 房东，房东要出租房子
public class Landlord implements Rent{
   public void rent() {
       System.out.println("房屋出租");
  }
}
```

```
//        System.out.println(method);
//        i++;
//        if ("rent".equals(method.getName())) {
//            seeHouse();
//            Object result = method.invoke(rent, args);
//            fare();
//            return result;
//        } else {
//            return method.invoke(rent, args);
//        }
```

```java
public class RentInvocationHandler implements InvocationHandler {
    private Rent rent;//这里Object也行
    public static int i;
    public RentInvocationHandler(Rent rent) {
        this.rent = rent;
    }
    //看房
    public void seeHouse(){
        System.out.println("带房客看房");
    }
    //收中介费 
    public void fare(){
        System.out.println("收中介费");
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        seeHouse();
        Object result = method.invoke(rent, args);
        fare();
        return result;

    }
}
```

```java
public class Client {
    public static void main(String[] args) {
        //new一个被代理类
        Landlord landlord = new Landlord();
        //代理实例的调用处理程序 RentInvocationHandler
        RentInvocationHandler pih = new RentInvocationHandler(landlord);
        Rent proxy = (Rent) Proxy.newProxyInstance(landlord.getClass().getClassLoader(),
                landlord.getClass().getInterfaces(), pih);//获取代理类
        proxy.rent();//代理实例执行 会走invoke
    }
}
```

> xxx.getClass().getClassLoader()。这里的xxx 我试了一下Object，Rent接口，handler，都可以
>
> xxx.getClass().getInterfaces()。这里的xxx 就是被代理类

> JDK动态代理打断点Debug与直接运行结果为什么不同？=》
>
> IDEA每debug一步就刷新toString方法，等生成代理类后，刷新代理类的toString方法时，会回调InvocationHandler的实现类的invoke方法

总结一下就是：

![image-20240521140937861](D:\picGo\images\image-20240521140937861.png)

- 1.首先我们用一个类去实现一个接口
- 2.再用一个类去实现InvocationHandler
- 通过 `Proxy.newProxyInstance(rent.getClass().getClassLoader(),
          rent.getClass().getInterfaces(),handler)`传入Rent接口的类加载器和接口，创建一个proxy，然后转型为Rent
- 此时Rent的实现类的toString，hashCode，equals，rent方法(重写)方法就被代理，当这些方法被调用时，会走handler的invoke，可以在method参数中拿到这些信息，args拿到每次method执行的参数，proxy就是代理类本身（一般不调用，会引起无限递归的栈溢出）



> jdk17无法看到ProxyGenerator了，它改成非public了



### CGLIB

CGLIB是一个功能强大，高性能的代码生成包。

当要**代理的类没有实现接口**或者为了**更好的性能**，CGLIB是一个好的选择

> Spring AOP的实现就是CGLIB

![image-20240521150305294](D:\picGo\images\image-20240521150305294.png)

我们来模拟一下cglib是如何完成代理的

![image-20240521151745899](D:\picGo\images\image-20240521151745899.png)

> enhancer先不用管，可以先理解为一个 cglib用来实现成为别的类的代理对象 的一个类就ok
>
> 它可以通过配置生成代理对象

#### QuickStart

**最简单的情况，cglib是如何完成代理**

1. 首先要有一个被代理类，**它没有实现任何接口**（区别jdk动态代理），有很多method

```java
public class TargetObject {
    public String method1(String paramName) {
        return paramName;
    }
 
    public int method2(int count) {
        return count;
    }
 
    public int method3(int count) {
        return count;
    }
 
    @Override
    public String toString() {
        return "TargetObject []"+ getClass();
    }
}
```

2.去实现一个`MethodInterceptor`，定义这个Interceptor执行的逻辑

```java
public class TargetInterceptor implements MethodInterceptor {
 
    /**
     * 重写方法拦截在方法前和方法后加入业务
     * Object obj为目标对象
     * Method method为目标方法
     * Object[] params 为参数，
     * MethodProxy proxy CGlib方法代理对象
     */
    @Override
    public Object intercept(Object obj, Method method, Object[] params,
            MethodProxy proxy) throws Throwable {
        System.out.println("调用前");
        Object result = proxy.invokeSuper(obj, params);
        System.out.println(" 调用后"+result);
        return result;
    }
}
```

> 可以看到在这之前加了一个`System.out.println("调用前");`
>
> 并用`proxy.invokeSuper`这里继续执行obj(被代理类实例)的方法，
>
> 执行完成后，再`System.out.println(" 调用后"+result);`，返回

> 这里的MethodInterceptor只会去拦截enhance设置的父类的方法。



3.创建一个enhancer，为其设置父类为被代理类，添加MethodInterceptor，并create一个proxy，向上cast为被代理类

```java
public class Main1 {
    public static void main(String[] args) {

        Enhancer enhancer =new Enhancer();
        //设置父类 为 被代理类
        enhancer.setSuperclass(TargetObject.class);
        //设置回调拦截器
        enhancer.setCallback(new TargetInterceptor());
        //生成代理类
        TargetObject targetObject2=(TargetObject)enhancer.create();
//        System.out.println(targetObject2);
//        System.out.println(targetObject2.method1("mmm1"));
        targetObject2.method1("mmm1");
//        System.out.println(targetObject2.method2(100));
//        System.out.println(targetObject2.method3(200));
    }
}
```

执行结果：

```text
调用前
 调用后mmm1
```

分析：

> 注意MethodInterceptor的Object result = proxy.invokeSuper(obj, params);
>
> 拦截到方法后，会去执行该enhancer的父类的该方法。

![image-20240521152744220](D:\picGo\images\image-20240521152744220.png)



> 和jdk动态代理差不多，这个是执行父类的，那个是提供classloader和interfaces，handler执行接口实现类的。

#### FastClass机制分析

#### 延迟加载对象

延迟加载就是指**表达式只在必要时才求值**，在mybatis，mysql缓存，前端页面加载都有很多实际用途

cglib提供了对代理对象的延迟加载。

https://www.runoob.com/w3cnote/cglibcode-generation-library-intro.html

> 注意：在生成一个proxy对象后，虽然并没有去访问它的属性出发加载，但此时也不是null（由于框架原因）

![image-20240521171918150](D:\picGo\images\image-20240521171918150.png)

#### 接口生成器InterfaceMaker

抽取某个类的方法生成接口方法

```java
InterfaceMaker interfaceMaker =new InterfaceMaker();
//抽取TargetObject类的方法 生成 接口方法
interfaceMaker.add(TargetObject.class);
Class targetInterface=interfaceMaker.create();
for(Method method : targetInterface.getMethods()){
	System.out.println(method.getName());
}
```



## 为什么 JDK 动态代理只能基于接口代理，不能基于类代理？

> 因为 JDK 动态代理生成的代理对象需要继承 `Proxy` 这个类，在 Java 中类只能是单继承关系，无法再继承一个代理类，所以只能基于接口代理。
>
> 人家就这样实现的
>
> **public** **final** **class** **UserServiceProxy** **extends** **Proxy** **implements** **UserService**

## JDK 动态代理和 CGLIB 动态代理有什么不同



参考：https://segmentfault.com/a/1190000011291179

https://juejin.cn/post/6974018412158664734#heading-15

https://juejin.cn/post/6844903744954433544#heading-3

https://www.cnblogs.com/aduner/p/14646877.html

# 反射

反射干了什么 => 程序在运行期可以拿到一个对象的所有信息。

为什么需要反射 => 反射是为了解决在运行期，对某个实例一无所知的情况下，如何调用其方法。

比如Class，field，constructor，parameter等等。

每一个类都有一个Class实例

而一个`Class`实例包含了该`class`的所有完整信息

具体都是些API：https://blog.csdn.net/Hellowenpan/article/details/107560498

# Spring AOP 和 AspectJ 有什么关联？

都是 AOP 的实现框架，AspectJ 更全面。

Spring AOP 整合 AspectJ 注解与 Spring IoC 容器

比 AspectJ 在spring中 的使用更加简单，支持 API 和 XML 的方式进行使用。**但仅支持方法级别**

# 简述 Spring AOP 自动代理的实现

Spring AOP 自动代理是通过实现 Spring IoC 中的几种 BeanPostProcessor 处理器，在 Bean 的加载过程中进行扩展，如果有必要的话（找到了能够应用于这个 Bean 的 Advisor）则创建 AOP 代理对象， JDK 动态代理或者 CGLIB 动态代理。

# 请解释 Spring @EnableAspectJAutoProxy 的原理？

# Spring Configuration Class CGLIB 提升与 AOP 类代理关系？

https://sourceforge.net/projects/cglib/Code Generation Library

maven









```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      	StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh"); 
		// 准备上下文进行刷新，重置属性和检查环境设置
      prepareRefresh();

      // 获取新的 Bean 工厂
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 准备 Bean 工厂，为 BeanFactory 配置上下文的类加载器、后处理器、忽略依赖等
      prepareBeanFactory(beanFactory);

      try {
         //子类对 Bean 工厂进行后处理。
         postProcessBeanFactory(beanFactory);
        // 启动 Bean 后处理步骤
         StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
         // 调用在上下文中注册的 Bean 工厂后处理器。
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册 拦截 Bean 创建的 后处理器。
         registerBeanPostProcessors(beanFactory);
         beanPostProcess.end();

         // 初始化消息源，用于国际化
         initMessageSource();

         // 事件多播器
         initApplicationEventMulticaster();

         // 初始化特定上下文子类中的其他特殊 Bean。
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // 初始化单例 Bean （不是懒加载的单例bean）
         finishBeanFactoryInitialization(beanFactory);

         // 发布相应的事件，表明上下文已经刷新完成。
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
          // 重置 Spring 核心中的常见内省缓存，因为可能不再需要单例 Bean 的元数据。
         resetCommonCaches();
         contextRefresh.end();
      }
    }
}
```
 
