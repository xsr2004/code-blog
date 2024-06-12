# bean销毁

在doCreateBean的最后，会注册DisposableBean

```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
    //判断是否需要销毁 && 单例
    //bean instanceof DisposableBean || 有销毁处理器
   if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
      if (mbd.isSingleton()) {
       
         registerDisposableBean(beanName, new DisposableBeanAdapter(
               bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
      }
      else {
         // A bean with a custom scope...
         Scope scope = this.scopes.get(mbd.getScope());
         if (scope == null) {
            throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
         }
         scope.registerDestructionCallback(beanName, new DisposableBeanAdapter(
               bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
      }
   }
}
```

判断是否需要销毁

```java
//判断是否需要销毁：有DestroyMethod方法||有DestructionAwareBeanPostProcessor
protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) {
   return (bean.getClass() != NullBean.class && (DisposableBeanAdapter.hasDestroyMethod(bean, mbd) ||
         (hasDestructionAwareBeanPostProcessors() && DisposableBeanAdapter.hasApplicableProcessors(
               bean, getBeanPostProcessorCache().destructionAware))));
}
```

## DestructionAwareBeanPostProcessor

DestructionAwareBeanPostProcessor接口是spring为bean提供的销毁阶段处理器，它提供了

- requiresDestruction：判断某一个Bean是否需要销毁
- postProcessBeforeDestruction：具体销毁要执行的逻辑

## @PreDestroy的实现

1.CommonAnnotationBeanPostProcessor的构造函数，设置了@PostConstruct和@PreDestroy。

```java
public CommonAnnotationBeanPostProcessor() {
        setOrder(Ordered.LOWEST_PRECEDENCE - 3);
        setInitAnnotationType(PostConstruct.class);
        setDestroyAnnotationType(PreDestroy.class);
        ignoreResourceType("javax.xml.ws.WebServiceContext");
}
```

2.当@PreDestroy的方法被执行时，它会先把method包装为LifecycleMetadata，然后用反射调用这个方法

```java
public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
   LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
   try {
      metadata.invokeDestroyMethods(bean, beanName);
   }
```



## 适配者模式

在doCreatBean中去注册DisposableBean时，用了Adapter

其中用到适配器模式，目的：兼容各种设置销毁方法的场景（比如通过实现DisposableBean接口、使用@PreDestroy注解方式…），最后统一调用org.springframework.beans.factory.support.DisposableBeanAdapter#destroy就可以了。


我们在定义一个Bean时，如果这个Bean实现了DisposableBean接口，或者实现了AutoCloseable接口，或者在BeanDefinition中指定了destroyMethodName，那么这个Bean都属于“DisposableBean”，这些Bean在容器关闭时都要调用相应的销毁方法。


![image-20240609210710043](D:\picGo\images\image-20240609210710043.png)

## 总结：

- bean在创建时，若这个bean是DisposableBean，会注册到DisposableBeans这个map中
- bean销毁发生的时机
  - bean的销毁发生在容器关闭（context.close，钩子关闭）或者 主动销毁 （手动执行bean.destrory()或者close方法）
- bean销毁的逻辑
  - 会清除缓存
  - 销毁该bean依赖的bean
  - 执行该bean的销毁前处理器的钩子方法（@preDestroy）
  - adapter会根据bean实现的接口，去if-else地区执行bean自己的销毁方法（destroy？close？bean定义的destroyMethods）
  - 执行该bean的销毁后处理器的钩子方法

- @PreDestroy注解
  - 该注解是javax.annoation，spring的CommonAnnotationBeanPostProcessor的构造函数中设置了@PostConstruct和@PreDestroy的类和优先级
  - 它的实现是通过InitDestroyAnnotationBeanPostProcessor的，这个类实现了DestructionAwareBeanPostProcessor，在《销毁前方法》去将method包装为LifecycleMetadata，并反射去invokeDestroyMethods。