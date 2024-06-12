# Spring startUp设计(二)：源码+debug方法论分析

本篇文章通过图解，源码debug，

介绍Spring startUp设计，

你将知道

- startUp是如何追踪和监控这些启动信息？
- 如何通过debug源码一个step执行时startUp都干了什么？demo全跟踪

## startUp是什么？

**它是一个spring Boot Actuator为了监控应用程序各步骤的性能数据的设计。**

**通过它可以获得约定步骤的性能数据（如耗时）**

> 戳这 startUp demo quickStart，你会了解开发者如何使用startUp来监控应用程序的各step

## startUp是如何追踪和监控这些启动信息

最简单地介绍一下关键类和相关接口

startUpStep：指某一个工作步骤。对于一个工作步骤，它有name（名字），id（编号），parentId（上一个step编号）

ApplicationStartup：应用程序的工作步骤，规定一个start方法。去做一些记录该step时的初始化。

StartupTimeline：对应向/actuator/startup发送请求响应的json数据，封装了每一个step和它的执行信息（可以理解为结果的封装类，/actuator/startup的restful api会将数据封装到这个对象并响应）

BufferingApplicationStartup：对ApplicationStartup的实现，在应用启动期间记录启动步骤，并将这些步骤缓存在内存中（最常用）。它有一些重要成员变量需要去事先知道。

- #start方法：在这里对每次进来的step进行初始化，比如编号自增，初始化开始时间
- #end方法：step已经结束时调用，记录该step已经结束的一些操作，比如设置该step.end为true，更新存在的step数量

> 这里没有说明ApplicationStartup的其他实现

## debug demo

**我们对springboot的启动进行debug，探讨spring如何监控environment-prepared这个step的。**

> 这个过程设计spring的事件处理机制 戳这 

### 第一部分：理解大概流程

首先我们来到SpringApplication#run方法这里。

也就是你main启动时肯定会调用的

```java
@SpringBootApplication
@MapperScan("com.example.shiyan5ssmintegrate.mapper")
public class Shiyan5SsmIntegrateApplication {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Shiyan5SsmIntegrateApplication.class);
        application.run(args);
    }
}

```

301行它会执行#prepareEnvironment，来执行Environment的准备。

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
```

调用后来到这里，`SpringApplication`#prepareEnvironment方法的 343 行

在这里，会去调用这里会调用SpringApplicationRunListeners#environmentPrepared方法，代码大概如下：

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
		//创建一个environment，已经创建ok
		//省略......其他操作
    	/////这里会调用SpringApplicationRunListeners#environmentPrepared方法。
343line		listeners.environmentPrepared(bootstrapContext, environment);
		//省略其他操作
		return environment;
	}
```

那么SpringApplicationRunListeners#environmentPrepared方法做了什么？

```java
void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
    doWithListeners("spring.boot.application.environment-prepared",
                    (listener) -> listener.environmentPrepared(bootstrapContext, environment));
}
//接下来走这里（同一文件）
private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction) {
    doWithListeners(stepName, listenerAction, null);
}
//这是重头戏
private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
                             Consumer<StartupStep> stepAction) {
    StartupStep step = this.applicationStartup.start(stepName);
    this.listeners.forEach(listenerAction);
    if (stepAction != null) {
        stepAction.accept(step);
    }
    step.end();
}
```

在`SpringApplicationRunListeners`#environmentPrepared方法中，它传了一个stepName，为`"spring.boot.application.environment-prepared"`。然后传了一个Consumer<StartupStep>，意思是想调用listener#environmentPrepared方法。

> 这里你需要知道SpringApplicationRunListener

我们重点看《重头戏》部分

在这个部分中：

- 它调用了一个applicationStartup.start，即开启了一个step。
- 然后listeners执行之前传的`(listener) -> listener.environmentPrepared(bootstrapContext, environment)`，这里的意思是listeners去消费environmentPrepared
- 最后step.end();

也就差不多是这样的：

<img src="D:\picGo\images\image-20240531204823716.png" alt="image-20240531204823716" style="zoom:33%;" />



> 开启一个step（start）->  执行这个step  ->  step.end
>
> 很纯粹啊有木有，**它甚至都不是aop面向切面用代理，就是直接耦合的。**

### 第二部分：那么如何执行step呢？

其实就是

```java
doWithListeners("spring.boot.application.environment-prepared",
				(listener) -> listener.environmentPrepared(bootstrapContext, environment));

//doWithListeners部分代码
private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
			Consumer<StartupStep> stepAction) {
		//开启一个名为stepName的step
    	//遍历listeners，每一个都去执行那个(listener) -> listener.environmentPrepared(bootstrapContext, environment)
		this.listeners.forEach(listenerAction);
		//121-123行代码这里先不看，对理解这个没啥关系
		//结束这个step
	}
```

它去调用environmentPrepared方法。

debug中点进去看，你会发现，EventPublishingRunListener（这是一个其实干了这么一件事）

```java
@Override
public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
      ConfigurableEnvironment environment) {
   this.initialMulticaster.multicastEvent(
         new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
}
```

> 其实就是multicastEvent了一个ApplicationEnvironmentPreparedEvent事件。
>
> 然后根据multicastEvent的实现，会去getApplicationListeners（获取当前系统所有listeners），然后去找匹配的监听器，去调用匹配监听器的#onApplicationEvent，去处理这个ApplicationEnvironmentPreparedEvent事件。
>
> spring事件机制最基本的流程

也就是这样的流程

<img src="D:\picGo\images\image-20240531210208545.png" alt="image-20240531210208545" style="zoom:33%;" />

> 必须说明的是，这里说例子只是在spring启动中，environment-prepared这个step如何执行的
>
> **不一定所有step都是这样执行的。**
>
> 但容器启动的过程中，几乎所有的step都是这样去执行，就是事件机制。
>
> 具体可以看SpringApplicationRunListeners类，它基本就是SpringApplication启动时的step所要执行时用到的方法。一个一个的，
>
> 比如starting，environment-prepared，context-prepared。
>
> 很清楚



### 第二部分：start和end干了什么

你现在肯定很纠结，**satrt和end都干了什么**。

为什么他们就可以拿到该step的执行时间，是怎么做的。

```java
//源码SpringApplicationRunListeners.java
private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction,
                             Consumer<StartupStep> stepAction) {
    StartupStep step = this.applicationStartup.start(stepName);//do thing？what
    this.listeners.forEach(listenerAction);
    if (stepAction != null) {
        stepAction.accept(step);//do thing？what
    }
    step.end();//do thing？what
}
```

> 首先去chatgpt一下：解析BufferingApplicationStartup类。
>
> 这里简单总结一下重要点

首先，在BufferingApplicationStartup中，

- idSeq：AtomicInteger，用于定义一个线程安全的step编号。
- startTime：它从clock#Instant获取，用于定义该step的初始时间
- current：当前BufferedStartupStep，他是一个AtomicReference<BufferedStartupStep>，标记当前step

具体可以看start和record定义。

```java
//BufferedStartupStep#start方法，初始化开始记录step的准备工作
public StartupStep start(String name) {
    int id = this.idSeq.getAndIncrement();
    Instant start = this.clock.instant();
    while (true) {
        BufferedStartupStep current = this.current.get();
        BufferedStartupStep parent = getLatestActive(current);
        BufferedStartupStep next = new BufferedStartupStep(parent, name, id, start, this::record);
        if (this.current.compareAndSet(current, next)) {
            return next;
        }
    }
}
//BufferedStartupStep#record方法，用于记录step的细节
private void record(BufferedStartupStep step) {
    if (this.filter.test(step) && this.estimatedSize.get() < this.capacity) {
        this.estimatedSize.incrementAndGet();
        this.events.add(new TimelineEvent(step, this.clock.instant()));
    }
    while (true) {
        BufferedStartupStep current = this.current.get();
        BufferedStartupStep next = getLatestActive(current);
        if (this.current.compareAndSet(current, next)) {
            return;
        }
    }
}
//BufferedStartupStep#end方法，设置该step结束，回调record
@Override
public void end() {
    this.ended.set(true);
    this.recorder.accept(this);
}
```

基本就是

- start的时候，记录一个开始时间startTime，并给一个序号idSeq，确保线程安全。
- record在end的时候回调，记录过滤器（毕竟有的stepName不需要记录，开发者可以设置，[见文档](https://springdoc.cn/spring-boot-actuator-startup/)），并把相关events（这就是个开头说的StartupTimeline里面的封装数据）。
- end的时候，去设置该step为结束，并回调record，补充step的信息。

最显著的就是**线程安全问题**。



**为什么他这老是做一些线程安全的事情。**

> 因为容器在启动中，会开线程池，会有很多线程去同时执行不同的step。
>
> **比如Thread1在执行spring.context.bean-factory.post-process**
>
> **Thread2在执行另一个bean的spring.beans.instantiate。**
>
> 二者并发。
>
> 那么我们要求最终呈现的统计结果，首先编号要对，应该是一个依次递增的id序列，其次是parentId，上一个step。去把它用一个对象封装出来（就是timeline的events[]），并用json返回

>  **这里涉及到多线程调试的问题**

一些小记

在我的应用中，对/actuator/startup发送请求

- spring.beans.instantiate 361次
- spring.context.beandef-registry.post-process 13次



关键断点如下：

![image-20240531203303472](D:\picGo\images\image-20240531203303472.png)

可自行模拟

## 总结：

https://blog.csdn.net/weixin_47410172/article/details/126161277

https://blog.csdn.net/hanxiaotongtong/article/details/107812976