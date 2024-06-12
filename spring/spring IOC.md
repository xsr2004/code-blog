# spring

开放式架构

#  你对 IoC 的理解

Inversion of Control，一种面向对象的编程思想。

传统模式：当我依赖一个对象，我需要创建并进行赋值，我才能用这个对象

IOC：将这一过程交由IOC容器，你只需用一个方式(注解,xml配置)即可使用这个对象

# 为什么需要 IoC

屏蔽构造细节

更应该关注的是对象的运用

而不是如何初始化和构造等细节。

#  IoC 和 DI 的区别？

DI，依赖注入，它是实现IOC的一种方式

IoC 容器启动或者初始化的时候，通过构造器、字段、setter 方法或者接口等方式注入相关依赖。如@AutoWire，@Resource

# 构造器注入和 Setter 注入

都是：将一个对象所依赖的其他对象 实例化后  注入到该对象中



构造器注入：在创建对象时将 它的依赖项 传递给对象的构造函数

setter注入，在创建对象后 通过对象的setter方法 将依赖项传递给对象

特点：

- 构造器注入：
  - 初始化该对象会传递所有必需的依赖项，保证了bean不变
  - 确保所有依赖项都被设置，不会为null，更利于维护和测试
  - 不能循环依赖
  - 有序注入，更利于review
- setter注入
  - 更灵活，但需要判null
  - 底层通过反射机制注入

接口回调注入，通过实现 Aware 接口

#  BeanFactory和ApplicationContext有什么区别？

我们更常用ApplicationContext，

- 功能上

  - BeanFactory提供了基础的接口设施：比如bean的定义，加载，获取，实例化，控制其生命周期等

  - ApplicationContext在BeanFactory的基础上，提供更完善的框架支持，如事件机制，环境和属性抽象，模块驱动，MessageSource，统一的资源文件访问，国际化的功能

- bean加载上

  - BeanFactory延迟加载，ApplicationContext一次性加载，各有好处

  



# Spring Bean 的生命周期

![image-20240523151515276](D:\picGo\images\image-20240523151515276.png)

- 实例化 Instantiation
  - 读取bean配置（xml，注解，spi），解析成BeanDefinition 对象（**resource的定位，BeanDefinition 的解析和注册**，发生在refresh#invokeBeanFactoryPostProcessor）
  - 实例化前置处理：调用InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation()前置处理（aop代理对象在这可以实例化）
  - getBean，createBean实例化（发生在finishBeanFactoryInitialization#preInstantiateSingletons，涉及ioc 对bean的实例化）
  - bean具体实例化：先getSingleton去缓存中查，再看该bean依赖的bean是否实例化，根据bean是单例/多例去具体实例化，调用实例化处理器接口的BeforeInstantiation方法给机会返回一个proxy(aop的实例化)，createBeanInstance通过构造函数+反射实例化bean，添加到该bean的工厂方法，之后进行属性赋值和初始化。
  - 实例化后置处理
- 属性赋值 Populate
  - 对bean的依赖进行赋值和注入：如CommonAnnotationXXX（@Resource、@PostConstruct、@PreDestroy）和AutowiredAnnotationXXX（@Autowired、@Value）实现（也就是InstantiationAwareBeanPostProcessor的postProcessProperties/value方法），找不到要注入的bean就去创建
  - 实例化后置处理
- 初始化 Initialization
  - 预处理：检查bean是否 Aware 接口类型，是则回调
  - 初始化前置处理：调用BeanPostProcessor#postProcessBeforeInitialization
  - 初始化开始：@PostConstruct 标注方法 -> InitializingBean#afterPropertiesSet() -> 自定义初始化方法
  - 初始化完成阶段：检查isSingleTon，做一些事情(todo)，这也是Spring 提供的一个扩展点
  - 初始化后置处理：调用BeanPostProcessor#postProcessAfterInitialization（aop代理对象在这初始化）
- 销毁 Destruction
  - 执行时期：ApplicationContext关闭 || 主动销毁某个 Bean
  - 执行销毁：@PreDestroy(销毁前) -> DisposableBean 接口的 Bean 的回调 -> destroy-method 自定义的销毁方法
- GC

> 属性赋值的postProcessProperties具体怎么做的，如何去实现的
>
> aop代理对象的实例化和初始化与bean的实例化初始化的联系，InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation()？

https://cloud.tencent.com/developer/article/2333457

https://juejin.cn/post/7075168883744718856#heading-1

https://juejin.cn/post/6844904065457979405

# Spring 应用上下文的生命周期

AbstractApplicationContext#refresh

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

## 刷新阶段 - ConfigurableApplicationContext#refresh()

- **prepareRefresh：设置相关属性，例如启动时间、状态标识、Environment 对象**

  - 初始化环境属性源。

    - **涉及Environment，propertySource相关设计**

    - 默认走GenericWebApplicationContext的initPropertySources(重写的)。然后用WebApplicationContextUtils#initServletPropertySources来封装默认的PropertySources。具体是这样的：

    - ```java
      public static void initServletPropertySources(MutablePropertySources sources,
            @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
      	//检查 propertySources （一般都不为null）
         Assert.notNull(sources, "'propertySources' must not be null");
          //去PropertySources 获取context的k-v，类似hashMap的put
         String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME;
         if (servletContext != null && sources.get(name) instanceof StubPropertySource) {
            sources.replace(name, new ServletContextPropertySource(name, servletContext));
         }
          //去PropertySources 获取config的k-v，类似hashMap的put
         name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME;
         if (servletConfig != null && sources.get(name) instanceof StubPropertySource) {
            sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
         }
      }
      //注意这里的ServletContext都是servlet的，是tomcat的servlet
      ```

    - ![image-20240524150613941](D:\picGo\images\image-20240524150613941.png)

    - 用户可以通过extends ClassPathXmlApplicationContext来重写AbstractApplicationContext#initPropertySources方法，如向environment添加kvhttps://blog.csdn.net/u013277209/article/details/109177452

    - PropertySource，PropertySources可遍历的PropertySource的interfece

    - Environment https://www.cnblogs.com/strongmore/p/16220520.html

    - 开发应用：通过PropertySource加载配置文件，并在源码中读取![image-20240524150816211](D:\picGo\images\image-20240524150816211.png)

    - Environment表示当前Spring程序运行的环境，主要管理profiles和properties两种信息。读取配置文件信息，如系统属性，文件配置，profile，占位符#{}https://www.cnblogs.com/strongmore/p/16220520.html

    - ```java
      @GetMapping("/2")
      public Object testEnvironment2() throws IOException {
          StandardEnvironment environment = new StandardEnvironment();
          Properties properties = PropertiesLoaderUtils.loadProperties(new ClassPathResource("application.yml"));
          PropertiesPropertySource propertySource = new PropertiesPropertySource("local", properties);
          environment.getPropertySources().addLast(propertySource);
          System.out.println(environment.getProperty("port"));//9998
          return null;
      }
      ```

    - 处理@Value注解的注入时，就是通过解析占位符来实现的，`这下看懂了`

    - spring提供多种配置源

  - 验证必需的属性是否已设置。

    - ```java
      //点这行代码进去
      getEnvironment().validateRequiredProperties();
      
      //对于ConfigurableEnvironment#validateRequiredProperties方法（下陈validate方法）
      //它可以走AbstractEnvironment#validate || AbstractPropertyResolver#validate方法
      //前者的话就是走AbstractEnvironment.propertyResolver。
      	@Override
      	public void validateRequiredProperties() throws MissingRequiredPropertiesException {
      		this.propertyResolver.validateRequiredProperties();
      	}
      	//这个propertyResolver在constructor会被赋值一个PropertySourcesPropertyResolver
      protected AbstractEnvironment(MutablePropertySources propertySources) {
      		//省略
      		this.propertyResolver = createPropertyResolver(propertySources);
      		//省略
      	}
      	protected ConfigurablePropertyResolver createPropertyResolver(MutablePropertySources propertySources) {
      		return new PropertySourcesPropertyResolver(propertySources);
      	}
      //而这个PropertySourcesPropertyResolver 是 extends AbstractPropertyResolver。而这个类在源码中没有去重写#validate
      //相当于还会走AbstractPropertyResolver#validate。深金，害我笑了一下。
      //也就是：
          @Override
      	public void validateRequiredProperties() {
      		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
      		for (String key : this.requiredProperties) {
      			if (this.getProperty(key) == null) {
      				ex.addMissingRequiredProperty(key);
      			}
      		}
      		if (!ex.getMissingRequiredProperties().isEmpty()) {
      			throw ex;
      		}
      	}
      //上面这段代码去检查this.requiredProperties。如果get不到就报错。
      //那么requiredProperties有些什么捏？
      //首先他是linkedHashSet
          private final Set<String> requiredProperties = new LinkedHashSet<>();
      //正常打断点调试发现requiredProperties就是null，也就是不需要任何必须变量
      //这里我们去继承AnnotationConfigServletWebServerApplicationContext
      //   去重写initPropertySources（这个方法会在refresh的开始，是spring提供给用户的一个配置数据源的扩展点）
      //    把一个名为MYSQL_HOST的RequiredProperties装载到Environment中
      public class CustomApplicationContext extends AnnotationConfigServletWebServerApplicationContext {
       
          @Override
          protected void initPropertySources() {
              super.initPropertySources();
              //把"MYSQL_HOST"作为启动的时候必须验证的环境变量
              getEnvironment().setRequiredProperties("MYSQL_HOST");
          }
      }
      //这样我们就要执行java命令时 执行命令java -DMYSQL_HOST=”192.168.0.1” -jar 加这个MYSQL_HOST=”192.168.0.1” k-v，否则给你抛出MissingRequiredPropertiesException异常。这也是mybatis和mysql-connector-j在你没有配置mysql配置信息（如url）时的做法。
      //参考：https://blog.csdn.net/luzhensmart/article/details/118187033
      ```

      ![image-20240524174018913](D:\picGo\images\image-20240524174018913.png)

      

      

      ![image-20240530120731401](D:\picGo\images\image-20240530120731401.png)

      > propertySource是一个属性资源，一般用的PropertiesPropertySouce其实就是xxx.properties文件的配置信息(内部是Map<Srting,Object>)
      >
      > String对应propertySource的name，Object对应propertySource的source
      >
      > 通过PropertyResolver的实现类（PropertySourcesPropertyResolver）去做基础解析(get/set)，通过ConfigurationPropertySourcesPropertyResolver去做高级解析(spilt符，配合binder高级类型转换/绑定)，后者广泛用于springboot
      >
      > 另外
      >
      > boot升级中，设计Binder类和ConfigurationPropertySources工具类(这是个final)用于将MutablePropertySources中的PropertiesPropertySource.source转换为开发者要的bean。
      >
      > 比如：
      >
      > ```java
      >  // 创建环境对象
      > ConfigurableEnvironment environment = new StandardEnvironment();
      > Properties properties = new Properties();
      > 
      > // 加载属性文件 读取到properties对象中
      > try (InputStream inputStream = ConfigurationPropertySourcesPropertyResolverExample.class.getClassLoader().getResourceAsStream("config.properties")) {
      >     if (inputStream == null) {
      >         throw new RuntimeException("Cannot find 'application.properties' in the classpath");
      >     }
      >     properties.load(inputStream);
      > } catch (IOException e) {
      >     e.printStackTrace();
      > }
      > 
      > MutablePropertySources propertySources = environment.getPropertySources();
      > // 添加属性到环境中
      > propertySources.addLast(new PropertiesPropertySource("customProperties", properties));
      > Binder binder = new Binder(ConfigurationPropertySources.from(propertySources));
      > //将propertySource.source的key前缀为myapp的信息 封装为 一个MyAppProperties对象
      > MyAppProperties myAppProperties = binder.bind("myapp", MyAppProperties.class).orElseThrow(() -> new RuntimeException("No value bound for 'myapp'"));
      > ```
      >
      > ```properties
      > ## config.properties
      > myapp.name=MyApplication
      > myapp.version=1.0.0
      > ```
      >
      > ```java
      > //MyAppProperties对象，封装配置信息，binder将配置信息转换为它
      > public static class MyAppProperties {
      >     private String name;
      >     private String version;
      > ```
      >
      > 上述例子就将source的key的前缀为“myapp”的propertySource 转换成一个 MyAppProperties对象（这可能有点绕，看一下源码就知道了）
      >
      > binder内部实现用的是递归

      ![image-20240524174025142](D:\picGo\images\image-20240524174025142.png)

    - ![image-20240524174753061](D:\picGo\images\image-20240524174753061.png)

    - > 这下看懂了

  - 存储和重置应用监听器。

  - 初始化早期应用事件的集合。

- 启动阶段 - ConfigurableApplicationContext#start()
- 停止阶段 - ConfigurableApplicationContext#stop()
- 关闭阶段 - ConfigurableApplicationContext#close()



earlyApplicationListeners初始化了12个![image-20240524180141580](D:\picGo\images\image-20240524180141580.png)

```
652 this.earlyApplicationEvents = new LinkedHashSet<>();
```

652 new了一个新的监听事件集合。



- 对于obtainFreshBeanFactory

```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();//获取一个刷新的bean工厂
```

首先

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   refreshBeanFactory();//刷新 Bean 工厂
   return getBeanFactory();
}
```

```java
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    // 检查是否已经刷新过。如果没有刷新过，将其标记为已刷新。
    if (!this.refreshed.compareAndSet(false, true)) {
        // 如果已经刷新过，抛出 IllegalStateException 异常，表示不支持多次刷新。
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    // 设置 Bean 工厂的序列化 ID。
    this.beanFactory.setSerializationId(getId());//默认application
}
```

```
prepareBeanFactory(beanFactory);
```





## 对于prepareBeanFactory(beanFactory);

```java
//下面是源码
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // Tell the internal bean factory to use the context's class loader etc.
   // 设置 Bean 工厂的类加载器
   beanFactory.setBeanClassLoader(getClassLoader());//将应用上下文的类加载器设置为 Bean 工厂的类加载器。这样，Bean 工厂可以使用该类加载器加载类。避免bean加载错误
   if (!shouldIgnoreSpel) {
      beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   }//为 Bean 工厂设置一个标准的 Bean 表达式解析器
   
   //为 Bean 工厂添加一个属性编辑器注册器，这个注册器可以在 Bean 属性赋值过程中使用环境变量。
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // Configure the bean factory with context callbacks.
    //为 Bean 工厂添加 ApplicationContextAwareProcessor 以处理实现 ApplicationContextAware 等接口的 Bean。此外，忽略特定的依赖接口以避免自动装配这些接口的实例。
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
    
   // 将这四个自动注入bean,允许在 Spring 容器中注入 对应类的 实例。（举例子）
   // 不知道这有什么用（很鸡肋啊，可能没看懂吧）
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
   // 注册早期后处理器 ApplicationListenerDetector
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // Detect a LoadTimeWeaver and prepare for weaving, if found.
   if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // Register default environment beans.
   // 注册一些默认系统环境beans：environment，systemProperties，systemEnvironment，applicationStartup
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
   if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
      beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
   }
}
```

总结：

在prepareBeanFactory(beanFactory)中，做一些事情

- 设置beanFactory的classloader，以确保bean的正常使用
- 给beanFactory添加一些依赖，例如spel的表达式解析器，属性编辑器（@value，xml,yml文件配置就是这个东西解决的），应用上下文处理器（用于给bean注入一些顶级bean）
- 添加事件的处理器，具体做法就是添加一个ApplicationListenerDetector，#postProcessAfterInitialization会把容器中的listener 都add到ApplicaitionContext
- 添加一些默认系统环境的bean，比如environment，applicationStartup等等。



下面

对每一行源码进行剖析

```java
//详解beanFactory.setBeanClassLoader(getClassLoader());
beanFactory.setBeanClassLoader(getClassLoader());
它干了什么？
getClassLoader()是ResourceLoader接口的方法，ResourceLoader
AbstractApplicationContext extends DefaultResourceLoader 
DefaultResourceLoader实现了ResourceLoader#getClassLoader()。并提供了对应实现。
如下：
public ClassLoader getClassLoader() {
		return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
	}
它会首先获取DefaultResourceLoader.this.classLoader，若不存在，用工具类去获取默认的classLoader
默认的classLoader的实现是这样的：
@Nullable
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
		//尝试获取当前线程的上下文类加载器
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// 没有线程上下文类加载器 ? -> 使用当前类的类加载器
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
					// 还没有？尝试获取系统类加载器 
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}
也就是说这里把AbstractApplicationContext的ClassLoader赋值给了beanFactory，
    它有什么用
//设置类加载器可以确保BeanFactory在加载类和资源时使用正确的类路径，避免出现ClassNotFoundException或NoClassDefFoundError。
```

![image-20240525210235210](D:\picGo\images\image-20240525210235210.png)



ConfigurableBeanFactory提供了配置Bean工厂的能力，包括Bean作用域、Bean后处理器和单例管理。

AutowireCapableBeanFactory提供了自动装配Bean的方法autowireBean





```java
//详解：
if (!shouldIgnoreSpel) {
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
}
//首先beanFactory.getBeanClassLoader()。通过ConfigurableBeanFactory的getBeanClassLoader，走它的在AbstractBeanFactory的实现，return this.beanClassLoader。
//beanClassLoader默认是去获取ClassUtils.getDefaultClassLoader()的
129 private ClassLoader beanClassLoader = ClassUtils.getDefaultClassLoader();//AbstractBeanFactory.java

//然后把这个classloader交给StandardBeanExpressionResolver。它里面具体定义了
69 ExpressionParser expressionParser;//StandardBeanExpressionResolver.java
//这里先不讨论ExpressionParser的设计了 todo 

//StandardBeanExpressionResolver是默认BeanExpressionResolver实现，这个BeanExpressionResolver设计为了支持解析和评估的表达式字符串（spel）（让xml中和注解的value中，通过表达式语言（如SpEL）去动态计算值）。
//见下面BeanExpressionResolver的例子
这代码的意思就是
//为 Bean 工厂设置一个标准的 Bean 表达式解析器
```

`BeanExpressionResolver`：#evaluate去解析和评估的表达式字符串（spel）

例如：

```xml
<bean id="exampleBean" class="com.example.ExampleBean">
    <property name="property" value="#{systemProperties['user.name']}" />
    BeanExpressionResolver会解析这个表达式并返回当前系统用户的名称。
</bean>
```

再例如：

```java
@Value("#{systemProperties['user.name']}")
private String property;
```

> `#{systemProperties['user.name']}` 是一个SpEL表达式，`BeanExpressionResolver`会解析这个表达式并将结果注入到`property`字段中。



ExpressionParser todo

> 感觉就是类似jsp和编译原理，用于识别一个序列



```java
//详解： 简单介绍干了什么：为 Bean 工厂添加一个属性编辑器 注册器，这个注册器可以在 Bean 属性赋值过程中使用环境变量。
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

ResourceEditorRegistrar：用于注册与资源（如文件、URL 等）相关的 `PropertyEditor`。

PropertyEditorRegistrar向外提供了#registerCustomEditors，允许开发者自定义的属性编辑器。默认实现是对Resource，ContextResource，File，URL等资源进行了属性编辑。

```java
@Override
	public void registerCustomEditors(PropertyEditorRegistry registry) {
		ResourceEditor baseEditor = new ResourceEditor(this.resourceLoader, this.propertyResolver);
		doRegisterEditor(registry, Resource.class, baseEditor);
		doRegisterEditor(registry, ContextResource.class, baseEditor);
		doRegisterEditor(registry, WritableResource.class, baseEditor);
		doRegisterEditor(registry, InputStream.class, new InputStreamEditor(baseEditor));
		doRegisterEditor(registry, InputSource.class, new InputSourceEditor(baseEditor));
		doRegisterEditor(registry, File.class, new FileEditor(baseEditor));
		doRegisterEditor(registry, Path.class, new PathEditor(baseEditor));
		doRegisterEditor(registry, Reader.class, new ReaderEditor(baseEditor));
		doRegisterEditor(registry, URL.class, new URLEditor(baseEditor));

		ClassLoader classLoader = this.resourceLoader.getClassLoader();
		doRegisterEditor(registry, URI.class, new URIEditor(classLoader));
		doRegisterEditor(registry, Class.class, new ClassEditor(classLoader));
		doRegisterEditor(registry, Class[].class, new ClassArrayEditor(classLoader));

		if (this.resourceLoader instanceof ResourcePatternResolver) {
			doRegisterEditor(registry, Resource[].class,
					new ResourceArrayPropertyEditor((ResourcePatternResolver) this.resourceLoader, this.propertyResolver));
		}
	}
```

并提供了这些被官方承认的资源属性编辑器xxxEditor。如pathEditor

![image-20240525214659159](D:\picGo\images\image-20240525214659159.png)

对于一些类型，spring提供了一些自定义customXXXEditor，以支持自定义属性编辑器。以满足不同的开发需求。

举个例子

```xml
<bean id="demoBean" class="springSourseAnalyes.BeanA" scope="singleton" init-method="init" lazy-init="true" >
    <property name="createTime" value="2018-08-22" /> -- yyyy-mm-dd的格式能被正确解析为date对象吗？报错：类型转换错误
    <property name="nameString" value="崔春驰" />
    <!-- <property name="b" ref="b" /> -->
/bean>
```

那么你可以去通过extends PropertyEditorSupport，去重写registerCustomEditors方法，添加一些自定义的属性开发需求

```java

public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar{
	
	@Override
	public void registerCustomEditors(PropertyEditorRegistry registry) {
        //设定"yyyy-MM-dd"格式的值会被解析为Date.class
		registry.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
	}
}
```

然后把这个bean去装载到CustomEditorConfigurer的propertyEditorRegistrars中，



CustomEditorConfigurer就可以扫描并执行我们重写的DatePropertyEditorRegistrar#registerCustomEditors方法了。

```java

<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
		<property name="propertyEditorRegistrars">
			<list>
				<bean class="springSourseAnalyes.DatePropertyEditorRegistrar" />
			</list>
		</property>
	</bean>
```

为什么要这样写？

```java
//CustomEditorConfigurer.java
@Nullable
private PropertyEditorRegistrar[] propertyEditorRegistrars;
```

> <list>DatePropertyEditorRegistrar</list>，其实就是把DatePropertyEditorRegistrar装在这个[]中

environment家族

![image-20240525212437709](D:\picGo\images\image-20240525212437709.png)

ConfigurableEnvironment通过PropertyResolver#getProperty去获取属性，还有getRequiredProperty



## 分析beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

### 添加应用上下文BeanPostProcessor

  //为 Bean 工厂添加 ApplicationContextAwareProcessor 以处理实现 ApplicationContextAware 等接口的 Bean。此外，忽略特定的依赖接口以避免自动装配这些接口的实例。

```java
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```



### BeanPostProcessor处理器

>  一句话描述：后置处理器，作用是在Bean对象在实例化和依赖注入完毕后，在**显示调用初始化方法的前后**添加我们自己的逻辑。



在spring应用上下文的#prepareBeanFactory方法中提到的

```java
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```

其实就是把一个ApplicationContextAwareProcessor处理器添加到beanFactory。



### ApplicationContextAware

ApplicationContextAware：应用上下文适配接口，实现它后可以注入ApplicationContext，在业务逻辑中使用ApplicationContext，可以做很多事情。

> 访问 Spring 上下文、发布事件、访问资源文件或进行国际化处理的 bean

比如下面的例子

```java
@Service
public class MyService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
	//注入方式2：和上面这个本质一样，只不过可能由那个AutoWiredxxx类来去setApplicationContext
    //@AutoWired
    //private ApplicationContext applicationContext;
    
    public void performTask() {
        // 从 ApplicationContext 获取另一个 bean
        AnotherService anotherService = applicationContext.getBean(AnotherService.class);
        anotherService.execute();
    }
}
```

上述代码中，我们定义了一个MyService，我们希望将ApplicationContext注入到该bean，因为在performTask方法中我们要通过ApplicationContext去#getBean。

对于ApplicationContext注入到MyService这个bean，谁处理的？ -> 其实就是ApplicationContextAwareProcessor

> ApplicationContextAwareProcessor：在spring启动后，当 Spring 容器中存在实现了 `ApplicationContextAware` 接口的 bean 时，`ApplicationContextAwareProcessor` 会在 bean 初始化之前（#postProcessBeforeInitialization）调用 `setApplicationContext(ApplicationContext applicationContext)` 方法，将当前的 `ApplicationContext` 注入到该 bean 中。
>
> 看一下源码就很清晰

### ApplicationContextAwareProcessor

如果这个bean需要注入ApplicationContext

spring的ApplicationContextAwareProcessor是如何将ApplicationContext注入到该bean的bean？

> 其实这要看它的源码内容：
>
> ApplicationContextAwareProcessor#postProcessBeforeInitialization在每一个bean的初始化前，将bean传入，判断类型，setApplicationContext。(见 ApplicationContextAwareProcessor 源码82)

```java
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    //	检查bean的类型，若是特定的这些顶级接口类型，直接返回（他们本身就是我们要的）
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
				bean instanceof ApplicationStartupAware)) {
			return bean;
		}
		//如果存在安全管理器（SecurityManager），则获取 AccessControlContext。这是为了在有安全限制的环境中，以特权模式执行某些操作。
    	//tip这一行不重要：（因为不管怎么样都要#invokeAwareInterfaces）
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}
		
		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
```

下面看一下invokeAwareInterfaces方法（源码ApplicationContextAwareProcessor|109行）：

```java
private void invokeAwareInterfaces(Object bean) {
		//other 处理
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
    	//other 处理
	}
```

这里去执行setApplicationContext方法，将你要的applicationContext注入



**为什么要`忽略特定的依赖接口以避免自动装配这些接口的实例。`**

```java
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
```

> 通俗一点来说，spring不希望你去注入ApplicationContextAware这种bean，因为它是spring的内部实现，很多bean是基于它的。
>
> 更希望你去实现这些顶级接口
>
> 比如
>
> ```java
> public class Bean {
>     @Autowired
>     private ApplicationContextAware applicationContextAware;//no，报错：required a single bean, but 20 were found:
>     @Autowired
>     private ApplicationContext applicationContext;//yes
> }
> ```



综上简单分析了BeanPostProcessor的设计，以及实现ApplicationContextAware接口的bean如何注入ApplicationContext的。

ApplicationContextAwareProcessor是把你要的ApplicationContext注入到特定bean的。

并顺带探讨了一下，为什么要`忽略特定的依赖接口以避免自动装配这些接口的实例。`



## 分析beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

**ApplicationListenerDetector会在#postProcessAfterInitialization方法（也就是bean初始化完成后），将实现ApplicationListener的Listener添加到applicationContext中。**

## spring事件处理机制

 Spring 应用上下文使用**事件驱动模型**来处理应用程序的状态变化

它的主要功能是检测 Spring 容器中的 `ApplicationListener` bean，并将其注册到 `ApplicationEventMulticaster` 中，以便能够接收和处理事件。

首先来看一个事件小demo

### spring 事件小demo

springboot默认启动会把AnnotationConfigServletWebServerApplicationContext注入ApplicationContext

在下面例子中，我定义了一个get接口，传一个String message，接收到参数后，把它包装成一个CustomEvent，并由ApplicationEventPublisher#publishEvent去发布它，由它的监听器去处理监听逻辑。







**applicationContext是如何根据注册的Listener，去对应好每一个事件进行处理的？换句话说，怎么监听的？**



> 发布事件后，会委托给ApplicationEventMulticaster#multicastEvent方法，该方法又负责将事件分派给所有匹配的监听器
>
> 所以我们就可以看#multicastEvent是怎么work的
>
> 下面是核心代码
>
> ```java
> @Override
> public void multicastEvent(final ApplicationEvent event) {
>  //传入一个事件
>  //找出匹配的listener
>  for (final ApplicationListener<?> listener : getApplicationListeners(event, eventType)) {
>      Executor executor = getExecutor();
>      if (executor != null) {
>          executor.execute(() -> invokeListener(listener, event));
>      } else {
>          invokeListener(listener, event);
>      }
>  }
> }
> ```
>
> getApplicationListeners(event, eventType)去筛选出 能够处理当前事件类型的监听器，把他们get出来
>
> ```java
> protected Collection<ApplicationListener<?>> getApplicationListeners(
>      ApplicationEvent event, ResolvableType eventType) {
> 
>  Object source = event.getSource();
>  Class<?> sourceType = (source != null ? source.getClass() : null);
>  //先查缓存	
>  ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
> 
>  ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
>  if (retriever != null) {
>      return retriever.getApplicationListeners();
>  }
> 
>  List<ApplicationListener<?>> allListeners = new LinkedList<>();
>     //去查所有ApplicationListener
>  for (ApplicationListener<?> listener : getApplicationListeners()) {
>      //支持该事件的listener添加到最终的allListeners
>      if (supportsEvent(listener, eventType, sourceType)) {
>          allListeners.add(listener);
>      }
>  }
>  return allListeners;
> }
> ```
>
> 而invokeListener又执行了listener.onApplicationEvent(event);来完成监听逻辑
>
> ```java
> protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
>  try {
>      listener.onApplicationEvent(event);
>  } catch (ClassCastException ex) {
>     //捕获到一些异常
>  }
> }
> ```
>



**在#getApplicationListeners中，是怎么做缓存处理的**

> 缓存数据结构的设计：
>
> ```java
> final Map<ListenerCacheKey, CachedListenerRetriever> retrieverCache = new ConcurrentHashMap<>(64);
> ListenerCacheKey：event的ResolvableType，event.source.class
> CachedListenerRetriever：Set<ApplicationListener<?>>和applicationListenerBeans（volatile）
> ```
>
> 查询的策略：先查retrieverCache，若里面没有要的listener，则去查所有的listeners，并做一些填充处理，再把对应信息写入retrieverCache。

```java
//retrieverCache数据结构，key是一个source和type复合，value是一个Set<ApplicationListener<?>>和Set<String>复合
Map<ListenerCacheKey, CachedListenerRetriever> retrieverCache = new ConcurrentHashMap<>(64);
从缓存中检索与特定事件类型和源类型对应的监听器集合。
如果缓存中没有现有条目，它会尝试创建一个新的缓存检索器并填充它（源码190 - 211）
然后跳到retrieveApplicationListeners去具体根据检索器来获取
    
在retrieveApplicationListeners中，根据检索器，去真正找到事件匹配的监听器，并去
这段debug好难，因为不懂检索器，容器完全运行起来，retriever还是空，它到底在这其中起到了什么作用....

ok！明白了
初始化后，retriever检索器会被“充满”，当下次请求过来时（如果这个请求触发了某个事件），那么会先从retriever去直接get它要的listener。
而不是去将所有的listener遍历，然后看这些listener是否能够处理该类型的事件（通过反射listener找泛型T）。提升了 当事件触发时，寻找对应listener的效率。
```

>  通过这种机制，Spring 能够有效地管理和分派事件监听器，确保在发布事件时能够快速找到并调用所有匹配的监听器

[Spring 泛型处理之 ResolvableType](https://blog.csdn.net/zzuhkp/article/details/107749148)

### earlyApplicationListeners存在的意义

ApplicationListeners不就可以了吗？为什么还要个earlyApplicationListeners？

> 记录容器初始化时监听器的状态：`earlyApplicationListeners` 保存了应用上下文在第一次初始化时的监听器列表。这是应用上下文最原始的监听器配置
>
> spring应用上下文或生命周期中，会经历多次刷新，需要去区分开  初始化 和 刷新后 的监听器状态
>
> 具体要去debug 感受一下 todo



## getApplicationListeners debug源码分析+感悟

**applicationContext是如何根据它的Listener，去对应好每一个事件进行处理的？换句话说，怎么监听的？**

> 回顾一下：
>
> 1.委托：其实ApplicationEventPublisher#publishEvent会将这个event**委托**给ApplicationEventMulticaster#multicastEvent处理（这是一个事件广播类）
>
> 2.寻找对应监听器：multicastEvent中，会getApplicationListeners。这个方法会先从内存的map中获取对应可处理该event的listener。若找不到则遍历容器所有的监听器，一个一个去supportsEvent检查，最终把这个`可处理事件的一些listener`去返回
>
> 3.回调监听器#onApplicationEvent：#invokeListener中会调用每一个listener#onApplicationEvent(event);



过一遍这个过程：

将一个断点挂在AbstractApplicationEventMulticaster.java文件的190行。

![image-20240528201434499](D:\picGo\images\image-20240528201434499.png)

容器加载启动环节：在debug中，这个环节有下面的事件相继触发，如ApplicationStartEvent，ApplicationEnvironmentPreparedEvent，ApplicationContextInitializedEvent，BootstrapContextClosedEvent，ApplicationPreparedEvent

#getApplicationListeners会把匹配这些事件的listener返回。

> 容器加载启动中，retrieverCache是null的，所以这个阶段会把这些事件和处理他们的监听器存入retrieverCache中。

举一个例子，如下

![image-20240528201924878](D:\picGo\images\image-20240528201924878.png)

比如ApplicationStartingEvent事件，这是容器启动的第一个事件，用来告诉所有组件 **”容器启动了“**

由于retrieverCache没东西，所以走221行，#retrieveApplicationListeners。

![image-20240528202309729](D:\picGo\images\image-20240528202309729.png)

**在#retrieveApplicationListeners中的247行，会对容器中初始的listener遍历，试图找这个ApplicationStartingEvent可以匹配的listener，把他们的对应关系写入retrieverCache，并返回这些“对ApplicationStartingEvent感兴趣”的listener**

![image-20240528202730932](D:\picGo\images\image-20240528202730932.png)

下图是容器刚开始（真刚开始）的listener情况，他们分别是：[见这篇文章]()。这涉及到SmartApplicationListener接口

![image-20240528202524691](D:\picGo\images\image-20240528202524691.png)

> 事实上，你会发现你写的自定义listener并没有加载，因为容器刚开始启动，bean（自定义listener bean）还没被依赖注入，初始化等等。

遍历后，我们发现有三个listener可以处理，分别是：

![image-20240528202648516](D:\picGo\images\image-20240528202648516.png)

最后把ApplicationStartingEvent事件和它对应的listener的匹配关系，存入retrieverCache的对应key的value（CachedListenerRetriever retriever）

![image-20240528202836742](D:\picGo\images\image-20240528202836742.png)

那么等下一次这个事件触发时，那就直接从retrieverCache就get出来了。

> 这里只是举例ApplicationStartingEvent，不代表这个事件还能再触发



ok，我们将容器整个运行完，并向其发送一个http请求，企图触发CustomEvent事件

![image-20240528203501566](D:\picGo\images\image-20240528203501566.png)

事件进来了

![image-20240528203453814](D:\picGo\images\image-20240528203453814.png)

我们看一下怎么运行的，由于第一次运行，关于CustomEvent事件和它的监听器对应关系没有存入retrieverCache中，所以第一次会去遍历所有listener（这里的listener就是全部的listener了，除了容器自带的，还有我们自定义的。

![image-20240528203714830](D:\picGo\images\image-20240528203714830.png)

此时已经有18个listener了，其中13,14是我们实现了ApplicationListener<CustomEvent>的listener。

遍历后，CustomEvent的监听器也找到了，

![image-20240528203820015](D:\picGo\images\image-20240528203820015.png)

将其写入retrieverCache中....（不填代码了）



ok，我们再来访问一次接口

这时retrieverCache已经存入了customEvent事件和它的对应监听器了，直接get出来返回即可。

![image-20240528204010141](D:\picGo\images\image-20240528204010141.png)



![image-20240528204101572](D:\picGo\images\image-20240528204101572.png)

> 这里还涉及到线程的问题，比如很多进程进来，如何去正确的getApplicationListeners（既不出错，也不能太慢）
>
> 还有的分析



一些联想：

debug时找断点，要找一个能贯穿上下文的，说明问题的断点。

debug时遇到不知道的一些类或者看不懂的流程啊，先看关不关键（影不影响你往下看），最好先gpt先看一下这个代码是做什么的，看一下源码javadoc中开发人员怎么说的。然后再分析

分析时应该最少知道它是怎么运行的；进阶一点要联系其他模块和顶层设计，理解为什么要这么设计，好处？解决的问题？根据问题去一遍一遍debug，并及时写一些小demo去说明问题，加深理解。

比如本篇中的retriever，retrieverCache的map设计，线程考虑，ClassUtils#isCacheSafe都是很值得思考的点。

知识点 还是 spring 的事件机制设计 ，高级点就是 事件订阅机制 ，软件工程 最基本的几种设计机制之一 。 



下面是 debug 中 的事件触发 

1.点击debug，启动容器

> ApplicationStartEvent
>
> ApplicationEnvironmentPreparedEvent
>
> ApplicationContextInitializedEvent
>
> BootstrapContextClosedEvent
>
> ApplicationPreparedEvent

2.web模块启动

> ServletWebServerInitializedEvent
>
> ContextRefreshedEvent
>
> ApplicationStartedEvent
>
> AvailabilityChangeEvent
>
> ApplicationReadyEvent
>
> AvailabilityChangeEvent

3.测试：发一个get请求

> CustomEvent（自定义）
>
> ServletRequestHandledEvent

4.测试：再发一个get请求

> CustomEvent（自定义）（这时，getApplicationListeners就走缓存了）
>
> ServletRequestHandledEvent





这个完了看一下todo

```java
singletonNames = new ConcurrentHashMap<>
```

todo：看一下ClassUtils.isCacheSafe，主要是ClassUtils的介绍，内容，用法，如何设计的。

todo：PropertyEditor自动发现机制https://www.cnblogs.com/yourbatman/p/14149463.html#propertyeditorregistry



平时看看https://java.jverson.com/basic/%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86.html

todo：HikariCP





## 分析 beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());

注册一些默认系统环境beans：environment，systemProperties，systemEnvironment，applicationStartup

```java
beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());//beans：environment
```

## registerSingleton解析

首先它会走DefaultListableBeanFactory.java（beanFactory默认实现）的registerSingleton

```java
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    //③：调用父类的registerSingleton
    super.registerSingleton(beanName, singletonObject);
    // ①处：这里更新一个手动注册的sigtonNames集合
    updateManualSingletonNames(set -> set.add(beanName), set -> !this.beanDefinitionMap.containsKey(beanName));
    //②处：这里清理了typeCache
    clearByTypeCache();
}
```

①处是怎么实现的？

```java
private void updateManualSingletonNames(Consumer<Set<String>> action, Predicate<Set<String>> condition) {
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
			synchronized (this.beanDefinitionMap) {
                //对set -> !this.beanDefinitionMap.containsKey(beanName)进行test，其实这跟manualSingletonNames没关系
				if (condition.test(this.manualSingletonNames)) {
                    //更新manualSingletonNames
					Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
					action.accept(updatedSingletons);//add一个bean
					this.manualSingletonNames = updatedSingletons;
				}
			}
		}
		else {
			// Still in startup registration phase
			if (condition.test(this.manualSingletonNames)) {
				action.accept(this.manualSingletonNames);
			}
		}
	}
```

> 总结一下①处how it works
>
> todo它通过action和condition，去更新manualSingletonNames（手动注册的单例bean名称）



然后把缓存清理一下：

```java
private void clearByTypeCache() {
   this.allBeanNamesByType.clear();
   this.singletonBeanNamesByType.clear();
}
```



ok，回过头看③，调用父类的registerSingleton

走的是DefaultSingletonBeanRegistry.java（SingletonBeanRegistry默认实现）

```java
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    Assert.notNull(beanName, "Bean name must not be null");
    Assert.notNull(singletonObject, "Singleton object must not be null");
    synchronized (this.singletonObjects) {
        //从singletonObjects去获取
        Object oldObject = this.singletonObjects.get(beanName);
        if (oldObject != null) {
            throw new IllegalStateException("Could not register object [" + singletonObject +
                                            "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
        }
        //应该就是要找不到，找不到，把singletonObject添加到单例bean中
        addSingleton(beanName, singletonObject);//分析见下
    }
}
protected void addSingleton(String beanName, Object singletonObject) {
    	//做一些map操作
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);//延迟创建单例对象，性能问题
			this.earlySingletonObjects.remove(beanName);//解决无限循环
			this.registeredSingletons.add(beanName);
		}
	}
```

> todo：earlySingletonObjects如何解决无限循环
>
> singletonFactories如何延迟创建单例对象，为啥
>
> 涉及知识点：spring三级缓存解决循环依赖



"org.springframework.boot.context.ContextIdApplicationContextInitializer$ContextId"

```java
static class ContextId {
   private final AtomicLong children = new AtomicLong();
   private final String id;
    //省略部分实现xxxx
}
```



## 对于postProcessBeanFactory(beanFactory);

执行AbstractApplicationContext|560行：

```java
postProcessBeanFactory(beanFactory);
```

会走**AnnotationConfigServletWebServerApplicationContext**。

下面是它的继承图（部分和debug有关的类进行类展示）

![image-20240530153437929](D:\picGo\images\image-20240530153437929.png)

下面是源码和相关注释

```java
//AnnotationConfigServletWebServerApplicationContext
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //调用父类 -> ServletWebServerApplicationContext#postProcessBeanFactory()方法
    super.postProcessBeanFactory()方法(beanFactory);
    //扫描一些东西，不重要
    if (!ObjectUtils.isEmpty(this.basePackages)) {
        this.scanner.scan(this.basePackages);
    }
    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}
```

ServletWebServerApplicationContext#postProcessBeanFactory()方法

```java
@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        //注册WebApplicationContextServletContextAwareProcessor，用于注入ServletConfigAware实现bean。
        //忽视依赖接口ServletContextAware(不让注入这个)
		beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
        //给beanFactory注册作用域
		registerWebApplicationScopes();
	}
```

注册完成后，beanFactory.scopes如下

![image-20240530151201071](D:\picGo\images\image-20240530151201071.png)

WebApplicationContextServletContextAwareProcessor被注册到beanFactory，这将让开发者获取到servletContext和servletConfig

![image-20240530153729793](D:\picGo\images\image-20240530153729793.png)

如：

```java
@Service
public class MyService {
    @Autowired
    private ServletContext servletContext;//注入ServletContext
    @Autowired
    private ServletConfig servletConfig;//注入ServletConfig
    public void performTask() {
        System.out.println(servletContext);
        System.out.println(servletConfig);
    }
}
```

## 对于StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");

```java
//记录和分析 Spring 应用程序启动过程中 "beans post-process" 阶段的性能
StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
// 调用在上下文中注册的 Bean 工厂后处理器。
invokeBeanFactoryPostProcessors(beanFactory);
// 注册 拦截 Bean 创建的 后处理器。
registerBeanPostProcessors(beanFactory);
//结束记录
beanPostProcess.end();
```

 `ApplicationStartup` 接口用于分析和记录应用程序启动过程，帮助开发者识别启动过程中的性能瓶颈和优化点。

start返回一个StartupStep实例，beanPostProcess.end();用来结束记录。可以记录各个步骤的耗时

> 是这个？笑了呀哥们

![image-20240530160631673](D:\picGo\images\image-20240530160631673.png)

this.applicationStartup？StartupStep？start？end？

记录的在哪？记录啥？

demo？怎么实现的？

怎么设计的？

### startUp简单demo

下面通过一个简单demo，介绍开发者如何quickStart。

添加actuator依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

启动类设置一个ApplicationStartup，

> 这里初始化为BufferingApplicationStartup，并设置最多处理步骤为1024（实际上spring整个启动都用不了500个）绰绰有余

```java
@SpringBootApplication
@MapperScan("com.example.shiyan5ssmintegrate.mapper")
public class Shiyan5SsmIntegrateApplication {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Shiyan5SsmIntegrateApplication.class);
        //设置一个BufferingApplicationStartup对象，最多处理步骤1024个
        application.setApplicationStartup(new BufferingApplicationStartup(1024));
        application.run(args);
    }
}
```

在yml中暴露/actuator端点

```yml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

通过http访问 http://xxx:port/actuator/startUp，得到下面的信息

<img src="D:\picGo\images\image-20240530193638860.png" alt="image-20240530193638860" style="zoom:33%;" />

> 记录了spring启动时各个步骤的耗时，包括是哪个startUpStep，duration等等
>
> tip：当然这个“各个步骤”指的是在代码中手动去start(process name)并end，

参考：https://www.amitph.com/spring-boot-startup-monitoring/#ApplicationStartup_StartupStep



DefaultTags内部类的设计

> 通过设计一个默认的 `DefaultTags` 实现并返回一个空的迭代器，Spring 提供了一个简洁、安全且易于维护的默认行为。这种设计模式可以避免空指针异常，简化代码逻辑，并确保接口契约的实现。它是面向对象设计中一种常见且有效的策略，用于处理可选或默认行为的情况。



addFilter测试demo

```java
@SpringBootApplication
@MapperScan("com.example.shiyan5ssmintegrate.mapper")
public class Shiyan5SsmIntegrateApplication {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Shiyan5SsmIntegrateApplication.class);
        //设置一个BufferingApplicationStartup对象，最多处理1024个步骤
        BufferingApplicationStartup applicationStartup = new BufferingApplicationStartup(1024);
        //添加过滤器，接收一个Predicate，用lambda表达式去只匹配"spring.boot.application.starting"的startupStep
        applicationStartup.addFilter(step -> step.getName().equals("spring.boot.application.starting"));
        application.setApplicationStartup(applicationStartup);
        application.run(args);
    }
}
```

![image-20240530202727200](D:\picGo\images\image-20240530202727200.png)



**分析BufferingApplicationStartup，成员变量定义，#start，#record**

#start方法分析：

```java
@Override
	public StartupStep start(String name) {
		int id = this.idSeq.getAndIncrement();//原子自增一个id
		Instant start = this.clock.instant();//通过clock获取一个startTime
		while (true) {
            // 迭代 StartupStep，返回当前新start的StartupStep
			BufferedStartupStep current = this.current.get();
			BufferedStartupStep parent = getLatestActive(current);
			BufferedStartupStep next = new BufferedStartupStep(parent, name, id, start, this::record);
			if (this.current.compareAndSet(current, next)) {
				return next;
			}
		}
	}
```

```java
private void record(BufferedStartupStep step) {
    	//检查过滤器和容量
		if (this.filter.test(step) && this.estimatedSize.get() < this.capacity) {
			this.estimatedSize.incrementAndGet();//原子增加estimatedSize
			this.events.add(new TimelineEvent(step, this.clock.instant()));//向events添加一个TimelineEvent
		}
    	//确保多线程环境下迭代正常
		while (true) {
			BufferedStartupStep current = this.current.get();
			BufferedStartupStep next = getLatestActive(current);
			if (this.current.compareAndSet(current, next)) {
				return;
			}
		}
	}
```

AtomicReference，AtomicInteger，ConcurrentLinkedQueue为什么用



金句：要理解一个接口，我们就去实现它的方法。





**StartupTimeline**

日志：debug后发现啥都没啊， 运行到571行，都beanPostProcess.end()了，什么都没有

![image-20240530160324684](D:\picGo\images\image-20240530160324684.png)





下图是567行registerBeanPostProcessors(beanFactory);调用前后的beanFactory.beanPostProcessors。从4个变成了9个，这很有意义。

我发现了一些比如AutoWired这样的老朋友。这需要去看

![image-20240530155953505](D:\picGo\images\image-20240530155953505.png)





## 解析：invokeBeanFactoryPostProcessors(beanFactory)

```java
//调用在上下文中注册的 Bean 工厂后处理器。
invokeBeanFactoryPostProcessors(beanFactory);
```





![image-20240601184557293](D:\picGo\images\image-20240601184557293.png)

112行invokeBeanDefinitionRegistryPostProcessors默认只会执行ConfigurationClassPostProcessor（BeanDefinitionRegistryPostProcessor）

而ConfigurationClassPostProcessor#processConfigBeanDefinitions

处理Spring应用上下文中的配置类（即标注有`@Configuration`的类），将其注册为Bean定义。这是Spring框架在启动时进行的一部分配置扫描和注册过程。

**第二部分：执行实现了ordered的BeanDefinitionRegistryPostProcessor**

> 由于上一步invokeBeanDefinitionRegistryPostProcessors时注册了一些新的BeanDefinitionRegistryPostProcessors
>
> 下面是执行再次查询结果：可以看到@MapperScanner的生效

![image-20240601182207213](D:\picGo\images\image-20240601182207213.png)

> 由于这俩各postProcessor都没添加到current里面（前者已经处理，后者没有实现ordered接口），所以在第二次的invokeBeanDefinitionRegistry并没有做任何事情。



***最后一步，执行没有实现PriorityOrdered或者Ordered的BeanDefinitionRegistryPostProcessor***



现在，去执行registry和regular的这些postProcessor，刚刚执行的是BeanDefinitionRegistryPostProcessor，现在去执行这些postProcessor的父类*BeanFactoryPostProcessor*方法

执行出来以后，发现又注册了这么多postProcessor

![image-20240601183642573](D:\picGo\images\image-20240601183642573.png)

**继续根据三个优先级去处理，**

然后再这次处理过程中，没有新的postProcessor注册进来。

清理缓存

```java
// 因为各种BeanFactoryPostProcessor可能修改了BeanDefinition
// 所以这里需要清除缓存，需要的时候再通过merge的方式获取
beanFactory.clearMetadataCache();
```

ok，这就是启动调用refresh#invokeBeanFactoryPostProcessors(beanFactory);实践debug



那么开发者可以通过去自定义实现BeanDefinitionRegistryPostProcessor/BeanFactoryPostProcessor的类，去在bean初始化依赖注入前后进行一些操作





问题来了：

- 为什么要设计BeanDefinitionRegistryPostProcessor，不能只有BeanFactoryPostProcessor吗？还设计为了父子类。

- 为什么处理3次后，就知道没有新的postProcessor注册，为什么不设计为循环呢？

  

参考：https://juejin.cn/post/6952901908847656991#heading-5

>  感觉先把代码大概注释找找，然后配合注释去debug，观察变量，过一遍，很不错。



## 解析：registerBeanPostProcessors(beanFactory);

刚才去invoke了beanFactoryPostProcessor。在这个过程中执行了BeanDefinitionRegistryPostProcessor和父类的BeanFactoryPostProcessor



### BeanFactoryPostProcessor和BeanPostProcessors的区别

这里要注意BeanFactoryPostProcessor和BeanPostProcessors的区别：

- BeanFactoryPostProcessor 是针对 BeanFactory 的扩展，主要用在 bean 实例化之前，读取 bean 的定义（Definition），并可以修改它。

- BeanPostProcessor 是针对 bean 的扩展，主要用在 bean 实例化之后，执行初始化方法前后。

结合bean生命周期那篇文章所说的  实例化 -> 属性赋值 -> 初始化 -> 使用 -> 销毁来讲。

>  BeanFactoryPostProcessor 在实例化前执行，BeanPostProcessor 在 bean 实例化之后，执行初始化方法前后执行。



### 回到registerBeanPostProcessors(beanFactory)干了什么

**就是将项目中所有实现了BeanPostProcessor的类都注册到beanFactory中。**

但是要注意：

> 它首先去获取这些实现了BeanPostProcessor的类的类名，获取到一个String[]中
>
> ![image-20240602163211301](D:\picGo\images\image-20240602163211301.png)
>
> 这时，这些类都没有被注册进去（这个方法过后就会被注册进去），而此时beanFactory真实的BeanPostProcessor只有这几个。
>
> 其中这4个都是在preparedBeanFactroy阶段去手动注册进去的，他们在系统的地位更高。
>
> ![image-20240602163335988](D:\picGo\images\image-20240602163335988.png)
>
> 等registerBeanPostProcessors结束，BeanFactroy就被充满了。
>
> ![image-20240602163529762](D:\picGo\images\image-20240602163529762.png)

### registerBeanPostProcessors(beanFactory)具体是怎么做的

```java
public static void registerBeanPostProcessors(
      ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

   // 1. 找出所有实现 BeanPostProcessor 接口的类
   String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    
   // BeanPostProcessor 计数
   int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
   // 2. 添加 BeanPostProcessorChecker , 主要用于记录信息到BeanFactory中
   beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

   //tip：这里还是和invokeBeanFactory的后置处理器一样，按照PriorityOrdered, Ordered，nonOrdered这样来分处理的优先级
    
    
   // 3. 定义不同的变量用于区分实现PriorityOrdered, Ordered 接口的 BeanPostProcessor 和普通的BeanPostProcessor
   // 3.1 priorityOrderedPostProcessors 存储实现PriorityOrdered 接口的BeanPostProcessor
   List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   // 3.2 internalPostProcessors 存储Spring内部的BeanPostProcessor
   List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
   // 3.3 orderedPostProcessorNames 存储实现Ordered接口BeanPostProcessor 的Name
   List<String> orderedPostProcessorNames = new ArrayList<>();
   // 3.4 存储普通的BeanPostProcessor 的BeanName
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   // 4. 遍历postProcessorNames
   for (String ppName : postProcessorNames) {
      // 4.1 实现PriorityOrdered 接口的BeanPostProcessor 处理
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
         priorityOrderedPostProcessors.add(pp);
         // 4.2 如果 pp对应Bean实例也实现了 MergedBeanDefinitionPostProcessor接口，则添加到internalPostProcessors
         if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
         }
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         // 4.3 实现了Ordered接口， 添加到 orderedPostProcessorNames
         orderedPostProcessorNames.add(ppName);
      }
      else {
         // 4.4 普通的 nonOrderedPostProcessorNames
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, register the BeanPostProcessors that implement PriorityOrdered.
   // 5. 首先， 注册实现了PriorityOrdered 接口的BeanPostProcessors
   // 5.1 排序
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   // 5.2 注册 registerBeanPostProcessors
   registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

   // 6. 接下来， 注册实现 Ordered 接口的BeanPostProcessors
   List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String ppName : orderedPostProcessorNames) {
      // 6.1 拿到ppName对应的 BeanPostProcessor
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      // 6.2 添加到orderedPostProcessors 准备执行注册
      orderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         // 6.3 如果pp实例也实现了 MergedBeanDefinitionPostProcessor接口， 添加到 internalPostProcessors
         internalPostProcessors.add(pp);
      }
   }
   // 6.4 排序
   sortPostProcessors(orderedPostProcessors, beanFactory);
   // 6.5 注册 orderedPostProcessors
   registerBeanPostProcessors(beanFactory, orderedPostProcessors);

   // 7. 注册普通的 BeanPostProcessors
   List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String ppName : nonOrderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      nonOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

   // 8. 最后，重新注册内部 internalPostProcessors，(相当于移动到链表的末尾)
   // 8.1 排序
   sortPostProcessors(internalPostProcessors, beanFactory);
   // 8.2 注册 internalPostProcessors
   registerBeanPostProcessors(beanFactory, internalPostProcessors);

   // 9. 重新注册 ApplicationListenerDetector，主要目的是移动到处理链的末尾
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}

```

> 总结一下：
>
> 1.先获取所有实现了BeanPostProcessor.class的类名
>
> 2.按照PriorityOrdered, Ordered，nonOrdered这样来分处理的优先级，将他们添加到各自的list中，sort排序后，add到beanFactory（保证优先级顺序



参考：https://juejin.cn/post/7000410020739284999



## 解析：initApplicationEventMulticaster

这个没啥，就是

- 先看xml配置中有没有对*id为aplicationEventMulticaster的bean对象*
  - 有，从bean工厂去得到它，并赋值给this.applicationEventMulticaster
  - 没有，applicationEventMulticaster将初始化为SimpleApplicationEventMulticaster，并registerSingleton。

```java
/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		//获取Bean工厂，一般为DefaultListBeanFactory
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//首先判断是否已有xml文件定义了id为aplicationEventMulticaster的bean对象
		//自定义的事件监听多路广播器需要实现AplicationEventMulticaster接口
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			//如果有，则从Bean工厂得到这个bean对象
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			//如果没有xml文件定义这个bean对象，那么新建一个SimpleApplicationEventMulticaster类作为aplicationEventMulticaster的Bean
			//因为SimpleApplicationEventMulticaster继承了AbstractApplicationEventMulticaster抽象类，而这个抽象类实现了aplicationEventMulticaster接口
			//因此SimpleApplicationEventMulticaster是aplicationEventMulticaster接口的一个实现
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}

```

```java
private void createWebServer() {
   WebServer webServer = this.webServer;
   ServletContext servletContext = getServletContext();
   if (webServer == null && servletContext == null) {
      StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
      ServletWebServerFactory factory = getWebServerFactory();
      createWebServer.tag("factory", factory.getClass().toString());
      this.webServer = factory.getWebServer(getSelfInitializer());
      createWebServer.end();
       //为webServer配置 管理 Web 服务器优雅关闭生命周期 的bean
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
       //为webServer配置 确保 web 服务器正常启动/关闭 的bean
      getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
   }
   else if (servletContext != null) {
      try {
         getSelfInitializer().onStartup(servletContext);
      }
      catch (ServletException ex) {
         throw new ApplicationContextException("Cannot initialize servlet context", ex);
      }
   }
   initPropertySources();//为什么这里又initPropertySources
}
```



## 解析：onRefresh

初始化主题源(themeSource)然后创建并启动WebServer。

>  SpringBoot内置的Tomcat或者UndertowWebServer就是在这里实例化的

### 初始化主题源(themeSource)

```java
this.themeSource = UiApplicationContextUtils.initThemeSource(this);
```

### 创建并启动WebServer

```java

private void createWebServer() {
		WebServer webServer = this.webServer;
    	//先获取ServletContext
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
            //获取WebServerFactory，这个值一般只有一个，因为服务器就一个，具体看getWebServerFactory 208行
			ServletWebServerFactory factory = getWebServerFactory();
			createWebServer.tag("factory", factory.getClass().toString());
            //根据获取WebServerFactory，去获取WebServer（这里涉及tomcat启动并将springboot context填充到tomcat中，看下文）
			this.webServer = factory.getWebServer(getSelfInitializer());
			createWebServer.end();
            //俩个registerSingleton，负责优雅/确保正常地管理web服务器的开启/关闭
			getBeanFactory().registerSingleton("webServerGracefulShutdown",
					new WebServerGracefulShutdownLifecycle(this.webServer));
			getBeanFactory().registerSingleton("webServerStartStop",
					new WebServerStartStopLifecycle(this, this.webServer));
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
    	//再初始化一次PropertySources（这里和之前preparedBeanFactroy#initPropertySources()不同
		initPropertySources();
	}
```

#### getWebServer干了什么

这里涉及到springboot如何启动webServer

本文这里的springboot内嵌了tomcat

>  设置host，connector，Engine，配置TomcatEmbeddedContext（添加父类classloader，DefaultServlet.....）并填充到host中。
>
> 然后初始化initialize（设置引擎名称，查找并配置上下文，启动 Tomcat守护进程

完成后在控制台输出tomcat的启动信息（如service，engine和协议，内嵌tomcat context正常初始化。

![image-20240602171448589](D:\picGo\images\image-20240602171448589.png)

![image-20240602171646154](D:\picGo\images\image-20240602171646154.png)

#### 这里和之前preparedBeanFactroy#initPropertySources()不同

![image-20240602172932045](D:\picGo\images\image-20240602172932045.png)

https://janus.blog.csdn.net/article/details/53895528

## 解析：registerListeners();

注册监听器

**具体做法是：将系统已有的listener这些bean注册到getApplicationEventMulticaster()中，把系统实现了ApplicationListener但还没有注册为bean的这些names注册到getApplicationEventMulticaster()的ApplicationListenerBeans**

![image-20240602173813414](D:\picGo\images\image-20240602173813414.png)



![image-20240602173950017](D:\picGo\images\image-20240602173950017.png)

> 比如上图中的customEventListener1/2就是开发者项目中自己实现的ApplicationListener
>
> 他们会被注册为listener的bean的，但not now。

并把早期事件发布

> ![image-20240602174105743](D:\picGo\images\image-20240602174105743.png)



参考：https://juejin.cn/post/7072257166874263583





## 解析：finishBeanFactoryInitialization

> 这个方法的作用是**初始化所有剩余的单例bean**
>
> 多么朴实无华，但在整个ioc容器里却是非常的牛逼

> 这个方法非常重要，涉及到很多知识点
>
> - spring在singleton的三级缓存，解决事件循环，结构清晰
> - AutowireCapableBeanFactory的doCreateBean方法（bean的实例化，非常之关键）
>

### 初识finishBeanFactoryInitialization

下面是finishBeanFactoryInitialization，配合详细的注释

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // 初始化conversionService（这是springboot提供的高级属性编辑器），这个时候去把conversionService注册获取bean并add到beanFactory
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }
   // 注册默认的嵌入值解析器
    //老实说已经注册了好多解析器，要把他们分个类，清晰一下每一个都是干什么的 todo spring解析器设计和注册的归纳分类
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // 初始化LoadTimeWeaverAware Bean实例对象
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   //冻结所有bean定义，因为马上要创建 Bean 实例对象了
   beanFactory.freezeConfiguration();

   // 实例化所有剩余（非懒加载）单例对象 （重点）
    //字面意思可以理解为 初始化Singletons前
   beanFactory.preInstantiateSingletons();
}
```

这里的#preInstantiateSingletons其实是**ConfigurableListableBeanFactory**接口定义的方法，**用于配置“初始化Singletons前”的一些行为**

> ConfigurableListableBeanFactory很熟悉了，基本都是它在办事

这里会走**DefaultListableBeanFactory**对ConfigurableListableBeanFactory的实现（只看有中文注释部分）

```java
@Override
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }
	
   //1.这里beanDefinitionNames拿到 所有bean的类名String
   //说明：beanNames里面存放的就是 所有的需要实例化的对象的全集,包含Spring自己的,和程序员自己添加的(Controller呀，component呀），还包含Aspectj的啥的
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   //2.根据每一个bean的beanName 去把这个bean注册出来
   for (String beanName : beanNames) {
       //2.1 合并它和它父类的bean定义信息 
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
       //2.2 Bean实例：不是抽象类 && 是单例 && 不是懒加载，否则不能被正常实例化
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
         //2.3 bean是否为FactoryBean
         if (isFactoryBean(beanName)) {
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               FactoryBean<?> factory = (FactoryBean<?>) bean;
               // 2.4 判断这个FactoryBean是否希望急切的初始化
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged(
                        (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  // 如果希望急切的初始化，则通过beanName获取bean实例
                  getBean(beanName);
               }
            }
         }
         //2.4 bean不是FactoryBean，正常getBean
         else {
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans...
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
               .tag("beanName", beanName);
         SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
         smartInitialize.end();
      }
   }
}
```



### **先来看getBean(beanName)发生了什么吧？**

首先，getBean会走 

```
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
```

**doGetBean很细节，很长，但结构很清晰。**

下面是具体代码注释（中文）

```java
@SuppressWarnings("unchecked")
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
      throws BeansException {
   //1. 将 name 真正解析成真正的 beanName，主要是去掉 FactoryBean 里的 “&” 前缀，和解析别名。
   String beanName = transformedBeanName(name);
   // bean会注册在这里
   Object beanInstance;

   // Eagerly check singleton cache for manually registered singletons.
   // 这里保留源码注释，意思就是 “急切地检查单例缓存中是否存在手动注册的单例”
    //2. 去this.Singletons里去getSingleton，赋值给sharedInstance （很重要的方法，有设计思想）（留坑1，getSingleton的实现）
   Object sharedInstance = getSingleton(beanName);
   // 3. 若getSingleton(beanName)不为null
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
       //3.1 直接根据sharedInstance去getObjectForBeanInstance，获取这个singleton的bean（这个看起来很牛逼，实际就是做了个bean?beanFactory的判断，很虚（留坑2：getObjectForBeanInstance的实现）
      beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);//这beanInstance和sharedInstance几乎没区别
   }
   //4. 若getSingleton(beanName)为null
   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      // 4.1 可能是处于“循环引用”中，判断一下
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }
	 // 4.2 接下来准备用BeanFactory去createBean（重头戏）
      BeanFactory parentBeanFactory = getParentBeanFactory();
      
       
       
	 // 由于parentBeanFactory一般就可以找到bean定义，而这一部分的逻辑主要就是：  如果找不到，应该xxx去创建bean
     // 所以省略掉
       
       
       
       
      StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
            .tag("beanName", name);
      try {
         if (requiredType != null) {
            beanCreation.tag("beanType", requiredType::toString);
         }
         // 合并它和它父类的bean定义信息 
         RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
          // 检查合并
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
          //就是检查该bean依赖的bean是否已经实例化，如果有就好，没有的话应该xxx
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               try {
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         // 4.3 判断mbd是否Singleton（一般都是）
         if (mbd.isSingleton()) {
             // 最重要的一集
             // 4.4 在getSingleton中去调用createBean，并把它的结果返回个sharediInstance （下面重点说 createBean部分）
            sharediInstance = getSingleton(beanName, () -> {
               try {
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }
		//4.4 如果mbd是Prototype 应该xxx
         else if (mbd.isPrototype()) {
            //省略
         }
		//4.5 如果mbd是其他 应该xxx （省略）
         else {
            String scopeName = mbd.getScope();
            //省略
      finally {
         beanCreation.end();
      }
   }
   //5. 用于将 Bean 实例适配为指定的类型。如果 Bean 的实际类型与所需类型不匹配，则尝试进行类型转换。
          //（实际写代码中基本都是匹配的）
   return adaptBeanInstance(name, beanInstance, requiredType);
}
```

**终于来到了#createBean方法，它是`AbstractBeanFactory`的抽象方法，**

**被spring官方评价为“有才华的”类：`AbstractAutowireCapableBeanFactory`实现，createBean方法是ioc框架的核心之一，它完成了实例化，属性赋值，初始化bean**

在createBean中会回调doCreateBean方法

### createBean解析

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
    // 1. 即将bean初始化，确保该beanClass已经被加载，（需要研究）
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName); 
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   // 2. 这里主要处理@Lookup注解，进行方法的替代！ 
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
       // 让 BeanPostProcessors 有机会返回代理而不是目标 bean 实例。
       // 2. 实例化前做一些事情（需要研究）
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
       // 3. 实例化bean （重点）
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
       //一般正常情况下这就返回了
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

> createBean方法进行实例化俩步走：实例化前dosomething，实例化

#### resolveBeanClass解析

#### resolveBeforeInstantiation解析

doCreateBean解析

doCreateBean的亮点line代码



通过instanceWrapper获取

```
Object bean = instanceWrapper.getWrappedInstance();
```

bean的属性赋值

```
populateBean(beanName, mbd, instanceWrapper);
```

初始化bean

```
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

调用bean的初始化前的BeanPostProcessors

```java
wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
```

调用bean的初始化方法

```
invokeInitMethods(beanName, wrappedBean, mbd);
```

调用bean的初始化后的BeanPostProcessors

```java
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
```



doCreateBean会了，ioc会50%了





看这个的感受：

如果想实例化,前提是不能是抽象类,不能是接口,非懒加载, 

**`AbstractAutowireCapableBeanFactory`**





https://www.cnblogs.com/ZhuChangwu/p/11755973.html

https://juejin.cn/post/7000792407260266527#heading-11





### **FactoryBean 和 BeanFactory的区别**

> **BeanFactory** 是 Bean 的工厂， ApplicationContext 的父类，IOC 容器的核心，负责生产和管理 Bean 对象。
>  **FactoryBean** 是 Bean，可以通过实现 FactoryBean 接口定制实例化 Bean 的逻辑，通过代理一个Bean对象，对方法前后做一些操作。

# spring 属性编辑器的设计

## 例子出发

> 例子转自：https://blog.csdn.net/u013277209/article/details/109201533

在spring中，我们可能有下面的需求，一个bean定义了一个属性，这个属性也是一个bean，我们要对他初始化。

如下：Person是一个类，它包含俩个属性，分别是String和Address对象。

```java
//Person部分
public class Person {
    private String name;
    private Address address;
//Address部分
public class Address {
    private String province;
    private String city;
    private String area;
```

现在我们在bean里想定义，将`"广东省-深圳市-南山区"`这样的String，根据`-`分隔符，转换为Address的province，city，area

这是我们的application-context.xml文件（spring配置）

```java
<bean id="person" class="com.bobo.customeditor.Person">
    <property name="name" value="bobo"/>
    <property name="address" value="广东省-深圳市-南山区"/>
</bean>
```

 如果直接这样做，它会报错：java.lang.ClassCastException。

**那么如何做到 在xml中的bean配置，把一个String赋值给一个property并成功转换呢？**

## 解决刚才的例子

**1.  自定义属性编辑器，编写转换规则**：我们去自定义一个自己的属性编辑器，它extends PropertyEditorSupport，重写#setAsText方法

并定义了将传入的text用`-`分割开来，赋给address的province，city，area，最后`setValue(address);`

```java
package com.bobo.customeditor;

import java.beans.PropertyEditorSupport;

/**
 * @author bobo
 * @date 2020-10-21
 */

public class AddressPropertyEdit extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        String[] split = text.split("-");
        Address address = new Address();
        address.setProvince(split[0]);
        address.setCity(split[1]);
        address.setArea(split[2]);
        setValue(address);
    }
}

```

**2.  注册这个编辑器：**

```java
public class AddressPropertyEditorRegister implements PropertyEditorRegistrar {
    @Override
    public void registerCustomEditors(PropertyEditorRegistry registry) {
        registry.registerCustomEditor(Address.class,new AddressPropertyEdit());
    }
}
```

高级应用（如数据绑定、BeanWrapper等）https://www.cnblogs.com/yourbatman/p/14149463.html#propertyeditorregistry

todo BeanWrapper

**3.  把编辑器要注册进去**

```java
<bean id="person" class="com.bobo.customeditor.Person">
        <property name="name" value="bobo"/>
        <property name="address" value="广东省-深圳市-南山区"/>
    </bean>
	
    <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
        <property name="propertyEditorRegistrars">
            <list>
                <bean class="com.bobo.customeditor.AddressPropertyEditorRegister"></bean>
            </list>
        </property>
    </bean>
—
```



为什么注册了就可以转换：

> PropertyEditorRegistry里面维护了Map<Class<?>, PropertyEditor> customEditors。它对应属性和类进行转换时使用到的属性编辑器
>
> 它会查找注册的 `PropertyEditor`
>
> 在bean初始化时（这是个spring ioc的核心），会AbstractAutowireCapableBeanFactory # docreatebean # populateBean #applyPropertyValues。
>
> 在这里，`BeanWrapper` 和 `BeanDefinitionValueResolver` 会自动查找并应用已注册的属性编辑器进行这种类型转换
>
> 这里有个源码逻辑的：



## 属性编辑器：它是怎么设计的？

![image-20240527170300847](D:\picGo\images\image-20240527170300847.png)

> PropertyEditor：属性编辑器，它定义了一些类型转换的顶级方法，如#setValue，#setAsText等方法。
>
> PropertyEditorSupport：实现了PropertyEditor，提供了更强大的功能，如listener默认配置
>
> CustomDateEditor....xxx：自定义属性编辑器，这是spring通过extends PropertyEditorSupport，来实现`数据绑定和类型转换`的需求。



> PropertyEditorRegistrar：注册器，提供#registerCustomEditor来注册属性编辑器，spring老xxxRegister了

PropertyEditorSupport一些文档

> https://pingfangx.github.io/java-tutorials/javabeans/advanced/customization.html
>
> 总的来说，本来是javaBeans曾为GUI设计的属性编辑器，然后Spring 框架拿它中用于数据绑定和类型转换的底层设计。

## 属性到底如何转换的？demo debug源码分析

UrlEditorTest去debug看setAsText是如何将传入的String转换为resource并存在value中的

Resource的介绍。

customEditors，customEditorsForPath，customEditorCache干啥？

bean加载时怎么调用注册的Editors，自动识别并转换为正确的对象的。

今天todo



## springboot的升级

Spring Boot 使用 Spring 4 引入的 `ConversionService`，替代传统的 `PropertyEditor`。`ConversionService` 提供了更灵活和强大的类型转换能力，支持更加复杂的类型转换需求。它可以处理基本类型、集合类型以及自定义的复杂类型转换。

```java
public class StringToDateConverter implements Converter<String, Date> {
    private final SimpleDateFormat dateFormat;

    public StringToDateConverter(String dateFormat) {
        this.dateFormat = new SimpleDateFormat(dateFormat);
    }

    @Override
    public Date convert(String source) {
        try {
            return dateFormat.parse(source);
        } catch (ParseException e) {
            throw new IllegalArgumentException("Invalid date format", e);
        }
    }
}

```

```java
@Configuration
public class MyConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultConversionService conversionService = new DefaultConversionService();
        // 注册自定义转换器
        conversionService.addConverter(new StringToDateConverter());
        return conversionService;
    }
}
```

它的好处是：线程安全，类型安全，boot生态集成

其他升级：

@ConfigurationProperties(prefix = "myapp")：用于将配置文件中的属性值绑定到 Java Bean 上

@Value：支持更复杂的 SpEL 表达式和类型转换，注入属性变得更加方便和灵活

> 当然springboot也支持PropertyEditorSupport等







# spring事件处理机制

Java中的Observer接口实践Observer模式

待看：https://www.cnblogs.com/xfeiyun/p/15797274.html



等把 prepareRefresh 看完



ApplicationEvent ，ApplicationListener<T>，ApplicationEventPublisher#publishEvent

# 类加载器



![image-20240525141528396](D:\picGo\images\image-20240525141528396.png)

![image-20240525142025457](D:\picGo\images\image-20240525142025457.png)

![image-20240525142518359](D:\picGo\images\image-20240525142518359.png)

![image-20240525144538598](D:\picGo\images\image-20240525144538598.png)



```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            //首先，检查该类是否已经加载过
            Class c = findLoadedClass(name);
            if (c == null) {
                //如果 c 为 null，则说明该类没有被加载过
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //当父类的加载器不为空，则通过父类的loadClass来加载该类
                        c = parent.loadClass(name, false);
                    } else {
                        //当父类的加载器为空，则调用启动类加载器来加载该类
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    //非空父类的类加载器无法找到相应的类，则抛出异常
                }       
    		if (c == null) {
                //当父类加载器无法加载时，则调用findClass方法来加载该类
                //用户可通过覆写该方法，来自定义类加载器
                long t1 = System.nanoTime();
                c = findClass(name);

                //用于统计类加载器相关的信息
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            //对类进行link操作
            resolveClass(c);
        }
        return c;
    }
}
```




# 类的生命周期

java源码是如何被jvm执行的

### 总流程

> 源码 -> 编译 -> 加载 -> 链接 ->初始化->使用 ->卸载

![image-20240525141542993](D:\picGo\images\image-20240525141542993.png)



### 源码 -> 编译 -> 加载

对于下面的java源码，先编译为.class文件，然后类加载器负责将.class文件装载到jvm内存中。

类加载器classLoader将.class文件装载到jvm内存中呢，主要做三件事情

- 1.根据.class文件的类名获取二进制字节流
- 2.字节流代表一种static存储结构，然后将其转化为runtime数据结构，存入方法区
- 3.java堆去生成一个该.class的Class对象（每一个被装载到方法区的.class文件里都会生成一个Class对象[参考🔗](https://java.jverson.com/jvm/java-reflection-class.html)），这个Class对象可以作为访问方法区对应runtime数据结构的入口。

如下图

![image-20240525145805398](D:\picGo\images\image-20240525145805398.png)

我们可以直接获取类名的class对象（方式很多：如getClass()对象，Class.forName()等，根据需要地去选择）

```java
Class<Person> personClass = Person.class;
```

我们可以拿这个Class对象，去创建实例，获取它实现的接口等等类信息。

```
Person person1 = personClass.newInstance();  // 创建类的实例（要求类有无参构造函数）
```



### 链接

回到类加载来，类加载后，jvm会干什么捏？

它首先会

- 验证：目的是确保Class文件的字节流中包含的信息符合当前虚拟机的要求，它做了文件格式验证，元数据验证，字节码验证，符号引用验证。

- 准备：为类的静态变量在方法区分配内存，并将其初始化为默认值

  > 这里的**初始化为默认值**，是指数据结构的默认值
  >
  > 比如 `public static int i=3`，在这个阶段i=0

- 解析：把类中的符号引用转换为直接引用

  > - **符号引用**是编译期生成的，与具体内存地址无关。它使用符号来表示类、字段或方法。
  > - **直接引用**是在运行时解析后的结果，指向具体的内存地址或对象实例。

上述过程称为链接

链接结束后，进行类的初始化。

### 类的初始化

初始化干了这几件事：

- 编译器会在将 `.java` 文件编译成 `.class` 文件时，收集所有类初始化代码和 `static {}` 域的代码，收集在一起成为 `<cinit>()` 方法
- 子类初始化时会先执行父类<cinit>()方法

### 卸载

**卸载类即该类的 Class 对象被 GC。**

卸载类需要满足 3 个要求:

1. 该类的所有的实例对象都已被 GC，也就是说堆不存在该类的实例对象。
2. 该类没有在其他任何地方被引用
3. 该类的类加载器的实例已被 GC



> 跟踪这个过程太麻烦了，后面再去一点点探索把，先就纯八股。



https://javaguide.cn/java/jvm/classloader.html#%E8%87%AA%E5%AE%9A%E4%B9%89%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8

https://pdai.tech/md/java/jvm/java-jvm-classload.html#jvm-%e5%9f%ba%e7%a1%80---java-%e7%b1%bb%e5%8a%a0%e8%bd%bd%e6%9c%ba%e5%88%b6

https://blog.csdn.net/ns_code/article/details/17881581

https://www.iteye.com/blog/zyjustin9-2092131



idea debug源码

# ClassUtils使用

spring是如何看待类，反射，加载器等问题的。

https://blog.csdn.net/wolfcode_cn/article/details/80660552

#  IOC容器初始化过程？

1. 从XML中读取配置文件。

2. 定位resource，将bean标签解析成 BeanDefinition，并注册到map中（springboot的该过程主要发生在refresh#invokeBeanFactoryPostProcessors()，spring发生在）

   > 在找到BeanDefinitionRegistryPostProcessor后，调用postProcessBeanDefinitionRegistry进行BeanDefinition的注册，然后走ConfigurationClassPostProcessor#processConfigBeanDefinitions去完成配置类(比如@Configuration)的BeanDefinition注册，在这个过程中用到parse去解析，走ConfigurationClassParser#doProcessConfigurationClass去处理@Import，ImportResource，ComponentScans，然后parse的doScan的findCandidateComponents去把resouce加载出来。

#  Bean注入容器有哪些方式

- @Configuration+@Bean
- @Componet的超集+@ComponetScan
- @Import注入
- @Configuration+@Bean注入一个自定义bean的factoryBean，然后去用工厂方法getObject注入这个bean，比如sqlSessionFactoryBean
- 实现**BeanDefinitionRegistryPostProcessor**，把你要注入的bean的class解析为一个beanDefinition并注册到容器中，容器在createbean时就可以找到这个bean的定义，进而创建bean（这个方法会在refresh#invokeBeanFactoryPostProcessors里调用）

# Bean的作用域

**singleton**：单例，Spring中的bean默认都是单例的。

2、**prototype**：每次请求都会创建一个新的bean实例。

3、**request**：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。

4、**session**：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP session内有效。

正常情况一般注入和使用的都是singleton，因为节省资源，对于配置、日志、数据库等操作好管理，性能也好

https://www.cnblogs.com/hello-shf/p/11051476.html 讲springboot自动装配的实现