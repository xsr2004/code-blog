# 解析：finishBeanFactoryInitialization

> 这个方法的作用是**初始化所有剩余的单例bean**
>
> 多么朴实无华，但在整个ioc容器里却是非常的牛逼

> 这个方法非常重要，涉及到很多知识点
>
> - spring在singleton的三级缓存，解决事件循环，结构清晰
> - AutowireCapableBeanFactory的doCreateBean方法（bean的实例化，非常之关键）

## 初识finishBeanFactoryInitialization

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



### **getBean(beanName)发生了什么吧？**

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

### createBean解析：开始实例化

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
	//省略部分源码

   RootBeanDefinition mbdToUse = mbd;
   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
    // 1. 即将bean初始化，确保该beanClass已经被加载，若没问题并添加到mbdToUse（需要研究）
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName); 
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   // 2. 这里主要处理@Lookup注解，进行方法的替代！ 
      mbdToUse.prepareMethodOverrides();
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    // 让 BeanPostProcessors 有机会返回代理而不是目标 bean 实例。
    // 2. 实例化前做一些事情（需要研究）
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
    // 4. 实例化bean （重点）
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    //一般正常情况下这就返回了
    return beanInstance;
   //下面是一些try-catch的catch，省略
}
```

> createBean方法做了什么：
>
> - 1.即将bean初始化，**确保该beanClass已经被加载**，若没问题并添加到mbdToUse
> - 2.**处理@Lookup注解**，进行方法的替代（这个先有个印象）
> - 3.**实例化前做一些事情**（让 BeanPostProcessors 有机会返回代理而不是目标 bean 实例。一般用于aop）
> - 4.**去实例化这个bean**（这是重点）

#### resolveBeanClass解析：该beanClass是否已被加载

我是不懂啊，虽然这里说`即将bean初始化，确保该beanClass已经被加载，`，但是我发现就算debug到下面这一行，

```java
return doResolveBeanClass(mbd, typesToMatch);
```

依然还是解析不出来什么东西，就是null。并且在createBeanInstance中也不影响什么，这个bean最终还是能被正常实例化。

所以我先不看这里了，因为它暂时没啥用



> todo ：beanClass对bean创建的作用问题

#### resolveBeforeInstantiation解析：bean实例化前做什么

在bean的实例化之前，检查扫描实现的`InstantiationAwareBeanPostProcessor`接口，并遍历这些BeanPostProcessor，去尝试获取一个bean（这个bean是个代理对象，可以理解为bean的增强版）

如果获取成功，就不会再往下的bean实例化了（因为已经有代替对象了）

> 这个`InstantiationAwareBeanPostProcessor`接口提供了在bean实例化之前的逻辑，并可以返回一个Object（这个Object就是自定义bean的实例化）
>
> 开发者可以通过自定义的bean实例化逻辑去实现自己的bean实例化。
>
> 例如AOP的代理对象生成就是在这里 具体可看https://juejin.cn/post/7242570945834074167

下面是resolveBeforeInstantiation源码分析

```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
   Object bean = null;
    //1. mbd还没有被实例化前解析过
   if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
      // 2. 如果mbd不是合成的 && 有实现InstantiationAwareBeanPostProcessor接口的PostProcessor
      if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
         Class<?> targetType = determineTargetType(beanName, mbd);
         if (targetType != null) {
             // 2.1 这里会遍历各PostProcessor#postProcessBeforeInstantiation方法，去尝试自己实例化这个mbd
            bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
            if (bean != null) {
                // 2.2 若bean真被实例化出来了，就去执行 实例化后 的逻辑
               bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
            }
         }
      }
      mbd.beforeInstantiationResolved = (bean != null);
   }
   return bean;
}
```

https://www.cnblogs.com/qianzhengkai/p/16257639.html

## doCreateBean解析：bean的实例化



```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
    //1. 有可能在本Bean创建之前，就有其他Bean把当前Bean给创建出来了（比如依赖注入过程中）。factoryBean需要重新创建
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
    //2. 初始化instanceWrapper：推断构造方法，@Bean的处理
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
    //3. 根据instanceWrapper获取bean 实例
   Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
    //4. 调用post-processors 允许他们 修改合并的bd
    //比如可以进行初始化方法的设置，比如添加属性值。spring有个默认的实现去寻找注入点
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }
  // 5.// 为了解决循环依赖提前缓存单例创建工厂
   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
    // 6. 第一次处理循环依赖
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // 7.初始化bean，进行属性赋值，初始化：包括三个回调以及初始化前中后
    //（具体看populateBean源码和InstantiationAwareBeanPostProcessor接口详解）
   Object exposedObject = bean;
   try {
      populateBean(beanName, mbd, instanceWrapper);
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }
  // 8. 第二次检查循环依赖 并处理
   if (earlySingletonExposure) {
     //省略
   }

   	// 9. 处理Bean销毁前要执行的方法
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }
  //10. 返回脱敏实例化，属性赋值，初始化后的bean
   return exposedObject;
}
```

> doCreateBean做了什么？
>
> - 1.初始化一个beanWrapper，并获取实例bean
> - 2.调用实现MergedBeanDefinitionPostProcessor的处理器，去添加自定义的beanDefinition合并后的逻辑（spring bean的扩展点）
> - 3.第一次处理循环依赖
> - 4.对实例化的bean进行属性赋值，初始化
>   - 若发现属性(一般是bean才会不存在) 不存在，则会去先创建这个属性
> - 5.第二次检查循环依赖 并处理

### BeanWrapper是什么？如何设计的？createBeanInstance和getWrappedInstance具体怎么做的？

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point.
   Class<?> beanClass = resolveBeanClass(mbd, beanName);
   // 2. 做一些校验：构造函数非public允许访问 && bean是public && beanClass解析成功
   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }
   // 3. 是否有bean的 Supplier 接口，如果有，通过回调来创建bean
   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }
	// 4. 如果工厂方法不为空，则使用工厂方法初始化策略
	// 通过 @Bean 注解方法注入的bean 或者xml 配置注入 的BeanDefinition 会存在这个值。而注入这个bean的方法就是工厂方法（重要）
   if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // 5.没有其他方式构造bean，spring即将通过构造函数去创建bean
   boolean resolved = false;//表示构造函数是否已经解析完成
   boolean autowireNecessary = false;//表示是否需要自动装配
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
          // 一个类可能有多个不同的构造函数，每个构造函数参数列表不同，所以调用前需要根据参数锁定对应的构造函数或工程方法
		// 如果这个bean的构造函数或者工厂方法已经解析过了，会保存到 mbd.resolvedConstructorOrFactoryMethod 中。这里来判断是否已经解析过了。

         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
    // 如果有已经解析过则使用功能解析好的构造函数方法，就用它来创建bean
   if (resolved) {
      if (autowireNecessary) {
          // 构造函数自动注入
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         return instantiateBean(beanName, mbd);
      }
   }

   // Candidate constructors for autowiring?
    // 6. 根据参数解析构造函数，并将解析出来的构造函数缓存到mdb 的  resolvedConstructorOrFactoryMethod  属性中
    // 到这一步，说明 bean 是第一次加载，所以没有对构造函数进行相关缓存(resolved 为 false)
    // 调用 determineConstructorsFromBeanPostProcessors  方法来获取指定的构造函数列表。后面详解
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // 获取最优的构造函数
   ctors = mbd.getPreferredConstructors();
   if (ctors != null) {
      return autowireConstructor(beanName, mbd, ctors, null);
   }

   // 使用默认构造函数构造
   return instantiateBean(beanName, mbd);
}
```

> 总结一下
>
> - 调用Suppilar供应商接口
> - 使用FactoryBean工厂方法
> - 构造函数的缓存判断
> - 构造函数实例化
>   - 有参/无参



### spring三层缓存是如何解决循环依赖的。

#### 什么是spring的三层缓存？

**三层缓存是spring 用于解决循环依赖的设计，**

所谓的三层缓存指的是：

- #### `singletonObjects` ，这是一个ConcurrentHashMap<String, Object>，它存储的是完全初始化的单例bean

- #### `earlySingletonObjects` ，这也是一个ConcurrentHashMap<String, Object>，它存储的是实例化但未初始化的单例bean（也有很多说法叫早期引用）

- #### `singletonFactories` ，这是一个HashMap<String, ObjectFactory<?>>，它存储的是用于创建这个bean的工厂

先来看一下ObjectFactory是什么东西？

> ObjectFactory是一个函数式接口（getObejct），定义如何获取这个bean
>
> 可以通过传入一个lambda表达式（javeSE知识）来说明getObject如何定义。
>
> 比如在下面bean创建的核心代码
>
> ```java
> public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
>    //这个函数是bean创建时会用到的核心代码之一
>     //它会先从一级缓存去获取这个beanName，若获取不到则去用singletonFactory函数式接口的getObject()去创建bean
>     //并提供了在创建单例bean的前置操作和后置操作
>     
>     //若获取不到则去用singletonFactory函数式接口的getObject()去创建bean
>     try {
>         singletonObject = singletonFactory.getObject();//这里就会调用createBean并拿到返回值。
>         newSingleton = true;
>     }
>     //这里意在先说明ObjectFactory。
>    }
> }
> ```
>
> ```java
> sharedInstance = getSingleton(beanName, () -> {
>       return createBean(beanName, mbd, args);
> });
> ```

#### spring到底是如何解决循环依赖的？

若有经典的A@AutowiredB，B@AutowiredA的情形

```java
//A.java
@Component("A")
public class A {
  @Autowired
  private B b;

  public A() {
      // 无参构造函数
  }
}
//B.java
@Component("B")
public class B {
  @Autowired
  private A a;

  public B() {
      // 无参构造函数
  }
}
```

> 首先spring不推荐这样的写法，因为循环依赖本身就是在业务中要极力避免的。

>  但spring是通过三级缓存去解决了这个问题的（顺便解决aop代理对象的问题）

> 若直接写这样的代码并允许，spring会报错，所以需要先设置

```yml
spring:
  main:
    allow-circular-references: true
```



先说结论：

- 当bean实例化后，会把这个bean的factory添加到三级缓存，后续会用这个factory去创建这个bean

- 当某一个bean用这个factory去创建出bean后，这个bean会被添加到二级缓存。

- 当初始化后，这个bean会被添加到一级缓存。

> 这么说太抽象，需要去debug感受一下这个过程

**下面我们来模拟A和B的实例化和初始化过程**

> 这里默认你是知道doCreateBean，populate，initializeBean，getSingleton（注意参数），addSingleton等核心方法的

> 自行debug这个过程，可以对照下面的梳理，完成整个模拟过程

**debug过程（文字描述）：**

A实例化，A存入三级缓存，属性赋值，注入时遇到B，去getBean(B)，走getSingle(B,true)，没找到（因为一切刚开始），去实例化B

B实例化，B存入三级缓存，属性赋值，注入时遇到A，去getBean(A)，走getSingle(A,true)，三级缓存里找到A，创建一个A（这个A实例化但没赋值没初始化），并把这个A添加到二级缓存，

上述创建出来的A，进行getObjectForBeanInstance直接返回到B的属性赋值了

B属性赋值结束，此时B属性赋值了一个A的早期引用。B此时应该是这样的

```
B@yyy{A@xxx{b=null}}
```

B初始化，第一次处理循环依赖，此时B初始化完成，回到getSingleton，将B添加一级缓存。

然后回到A的属性赋值，此时getBean(B)ok了，并且这个B在一级缓存，然后A**递归**里面所有属性进行属性赋值。A此时应该是这样的

```
A@xxx{B@yyy{a@xxx{B@yyy{....}}}}
```

A初始化，第一次处理循环依赖，回到getSingleton，将A添加一级缓存。

AB初始化结束。



**debug过程（图解描述）：**

![image-20240605141611242](D:\picGo\images\image-20240605141611242.png)

>  https://cloud.tencent.com/developer/article/1497692



#### 一些设计缓存之间转换的重要函数解析

##### getSingleton逻辑

- 在获取一个bean时，先从singletonObjects（一级缓存）获取，
  - 获取到则返回
  - 获取不到，若该bean已经被创建（这意味着该bean的早期引用在缓存）
    - 从earlySingletonObjects获取
      - 若获取不到，并且允许bean早期引用(一般为true)
        - 那么还是尝试从singletonObjects去获取一次，然后从singletonFactories去获取该bean的ObjectFactory
        - 调用ObjectFactory.getObject()方法（一般是个lambda表达式，为createBean方法）获取该bean

> 值得注意的是，在bean初始化后，AbstractAutowireCapableBeanFactory#633时，getSingleton的第二个参数为false
>
> 表明只能从一级缓存和二级缓存获取。这里要知道为什么这样做



##### addSingletonFactory逻辑

#addSingletonFactory会在bean实例化后被调用，它将把这个bean的factory存入三级缓存，用于解决依赖循环

```java
//核心用法：实例化bean后，把这个bean的工厂添加到三级缓存
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
synchronized (this.singletonObjects) {
   if (!this.singletonObjects.containsKey(beanName)) {
      this.singletonFactories.put(beanName, singletonFactory);
      this.earlySingletonObjects.remove(beanName);
      this.registeredSingletons.add(beanName);
   }
}
```

#### 为什么俩级缓存不行，偏偏要三层缓存

> 如果这个bean没被aop切面代理，只需一级缓存和三级缓存即可实现依赖循环
>
> 但如果这个bean被AOP进行了切面代理，每次从三级缓存中拿到singleFactory对象，执行getObject()方法又会产生新的代理对象，这是不行的，我们希望每次去执行getObject()方法都是同一个代理对象（单例的）



### populateBean(beanName, mbd, instanceWrapper);做了什么？

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  
   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
    //实例化后，属性设置之前。循环调用每个实现postProcessAfterInstantiation的方法
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
         if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
            return;
         }
      }
   }
	//重点：获取容器在解析Bean定义资源时为BeanDefiniton中设置的属性值
	//这个是程序员在 bd中 写入的属性rootBeanDefinition.getPropertyValues().add("type","男的");
   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
	// 依赖注入的逻辑 选择
   int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    // 这个判断是spring自带的依赖注入逻辑。可以在@Bean上配置autowire属性配置ByName或者ByType（已过期），现在是@Autowired和@Resource了
   if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // Add property values based on autowire by name if applicable.
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // Add property values based on autowire by type if applicable.
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

   PropertyDescriptor[] filteredPds = null;
    //需要依赖注入
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
          // 这里会调用AutowiredAnnotationBeanPostProcessor的postProcessProperties()方法，会直接给对象中的属性赋值。真正的处理@Autowired、@Resource、@Value注解
         PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
          //若这里为null，
         if (pvsToUse == null) {
            if (filteredPds == null) {
               filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               return;
            }
         }
         pvs = pvsToUse;
      }
   }
    // 判断需要属性注入
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }
// 如果当前Bean中的BeanDefinition中设置了PropertyValues，那么最终将是PropertyValues中的值，覆盖@Autowired
   if (pvs != null) {
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

> **属性赋值会做什么？**
>
> - 调用bean实例化后置接口的钩子
> - 校验bean是否实例化
> - 依赖注入的逻辑 选择
> - 对依赖注入进行处理
>   - 调用所有的InstantiationAwareBeanPostProcessor的postProcessProperties方法，实际处理@Autowired、@Resource、@Value注解
>   - 若没结果，则调postProcessPropertyValues处理
> - 对属性注入进行处理

**postProcessPropertiesValue不好吗？为什么要postProcessProperties**

> 这是spring 5的改版，先调前者的目的在于高效封装和直接（，调用后者则可以操作PropertyDescriptor[] pds，更加个性化毕竟crud程序员一般是用不到的。

**依赖注入和属性注入**

> 当时懵了一下属性注入是个啥，其实就是@value 把xml或者yml等配置文件的property，注入到bean中







### initializeBean(beanName, exposedObject, mbd);做了什么？

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
		   //如果有安全管理器（security的东西，省略）
            //和ioc无关，省略这部分代码
		}
		else {
            //调用Aware接口方法：其实就是三个：BeanNameAware，BeanClassLoaderAware，BeanFactoryAware。
			invokeAwareMethods(beanName, bean);
		}
		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            //1. 调用 bean初始化 前 处理器
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            //2. 初始化 @postconstruct 执行init-method方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            //3. 调用 bean初始化 后 处理器（若有aop，则在这创建代理对象并添加到容器中）
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

> initializeBean方法做了什么？
>
> 完成了bean的初始化操作：
>
> - 1.调用有关的Aware接口方法：填充bean的beanName，BeanClassLoader，BeanFactory
> - 2.调用 bean初始化 前 处理器
> - 3.调用 初始化 方法（@postconstruct -> InitializingBean#afterPropertiesSet（例子是sqlSessionFactoryBean在初始化后调用它来初始化配置） -> init-method
> - 4.调用 bean初始化 后 处理器（若有aop，则会在这创建代理对象并添加到容器中）





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



强烈推荐https://juejin.cn/post/7075168883744718856

这篇文章可以先看一遍留个印象。等信息和思考差不多后再去看，会很有感悟，



https://blog.csdn.net/qq_46144673/article/details/127906360

https://www.cnblogs.com/ZhuChangwu/p/11755973.html

https://juejin.cn/post/7000792407260266527#heading-11

https://juejin.cn/post/7069038008468504590#heading-1



## **FactoryBean 和 BeanFactory的区别**

> **BeanFactory** 是 Bean 的工厂， ApplicationContext 的父类，IOC 容器的核心，负责生产和管理 Bean 对象。
> **FactoryBean** 是 Bean，可以通过实现 FactoryBean 接口定制实例化 Bean 的逻辑，通过代理一个Bean对象，对方法前后做一些操作。

https://www.cnblogs.com/strongmore/p/16223021.html

比如SqlSessionFactoryBean的设计，容器刷新中对于bean的实例化流程，和三级缓存之间的设计，都和FactoryBean 息息相关



## debug一个例子

把bookController注入一个bookServiceImpl，去debug整个流程

## bean的生命周期

![image-20240605153051617](D:\picGo\images\image-20240605153051617.png)





## 看源码的感受和收获：

如果想实例化,前提是不能是抽象类,不能是接口,非懒加载, 

如何debug，debug在项目中的应用



https://www.51cto.com/article/702892.html

# 解析：finishRefresh

清除当前 Spring 应用上下文中的缓存，例如通过 ASM（Java 字节码操作和分析框架）扫描出来的元数据，并发布上下文刷新事件

```java
protected void finishRefresh() {
   // Clear context-level resource caches (such as ASM metadata from scanning).
    // 1.清理资源缓存
   clearResourceCaches();

   // Initialize lifecycle processor for this context.
    // 2.初始化lifecycle处理器
   initLifecycleProcessor();

   // 3.启动所有lifecycle有关的bean
   getLifecycleProcessor().onRefresh();

   // 4.封装contextRefresh事件，并发布出去，让广播器进行广播
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   if (!NativeDetector.inNativeImage()) {
      LiveBeansView.registerApplicationContext(this);
   }
}
```

**clearResourceCaches干了啥？**

> 清除ResourceCache
>
> 这个`ResourceCache` 是一个缓存机制，用于缓存 Spring 应用上下文中加载的资源，例如配置文件、静态文件等
>
> debug发现就是java源文件它给缓存了一下
>
> ![image-20240605205548315](D:\picGo\images\image-20240605205548315.png)

**initLifecycleProcessor干了啥？**

> 去获取beanName为“lifecycleProcessor”的bean，获取不到则new 一个DefaultLifecycleProcessor，给set到BeanFactory

**关于spring对LifecycleProcessor的默认实现DefaultLifecycleProcessor**

> 在onRefresh中，DefaultLifecycleProcessor会获取所有的lifecycleBeans，并对他们进行阶段分组为LifecycleGroup，并分别`LifecycleGroup::start`。（start方法是lifecycle里的，由这些lifecycleBeans去自行实现，一般就是this.running标记为true）

**finishRefresh干了什么**

> 清理环境中资源缓存，启动lifecyclebeans，发布contextRefreshed事件并广播



https://juejin.cn/post/6943025527619846152

https://juejin.cn/post/6940824556483379237

# populate详细说明

先说八股

- 提供机会在属性填充之前修改bean，（检查并调用bean实例化后处理器）
- 获取root bd的自定义Property（默认为空，程序员可以手动添加的）
- 选择spring bean自动配置模式（byName||byType）
- 检查bean实例化后处理器，若存在根据他们的postProcessProperties方法去属性赋值
- 在上述的属性赋值中：postProcessProperties中会查找注入bean的元数据( #findAutowiredMetadata )，并用metadata.inject，然后到element.inject，最后调用field.set完成属性赋值，并把结果真正设置到bw(beanWrapper）中（applyPropertyValues)



在resolveFieldValue的

```java
value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
```

https://jingzh.blog.csdn.net/article/details/131033983

https://blog.csdn.net/aqin1012/article/details/128800736

https://www.jianshu.com/p/20cf0116c5c0

# @Autowired和@Resource的区别？

- 来源不同，默认注入方式不同

  - @Autowired是spring内置的注解，默认按照byType来注入，也就是接口类型匹配要注入的bean
  - @Resource是j2ee内置的注解，默认按照byName来注入，也就是名称匹配要注入的bean，

  > 当一个接口存在多个实现类的情况下，`@Autowired` 和`@Resource`都需要通过名称才能正确匹配到对应的 Bean。`Autowired` 可以通过 `@Qualifier` 注解来显式指定名称，`@Resource`可以通过 `name` 属性来显式指定名称

- 作用位置不同
  - @Autowired可以用在字段，构造函数，方法，参数
  - @Resource则不可以在构造函数和参数上用。

# Bean的线程安全，如何实现bean的线程安全

几乎所有场景的 Bean 作用域都是使用默认的 singleton，在singleton下，因为单例，存在线程安全问题

多例则不存在

不过，大部分 Bean 实际都是无状态（没有定义可变的成员变量）的（比如 Dao、Service）

对于有状态单例 Bean 的线程安全问题，常见的有两种解决办法：

1. 在 Bean 中尽量避免定义可变的成员变量。
2. 在类中定义一个 `ThreadLocal` 成员变量，将需要的可变成员变量保存在 `ThreadLocal` 中（推荐的一种方式）。比如security存一个用户信息到这个request的线程中

# @Component 和 @Bean 的区别是什么？

- 作用位置不同
  - @Component在类上，@Bean在方法上
- 工作机制不同
  - @Component是通过扫描类路径（搭配@ComponentScan），看有哪些需要装载的bean，然后把他们装配到spring中
  - @Bean则是在配置类中使用，在java代码中定义如何返回这个bean，spring会把这个返回值注册成一个bean
- 前者用在正常的mvc架构中更好，后者一般用于自定义去装配第三方库的类，把他们注册成bean，比较灵活