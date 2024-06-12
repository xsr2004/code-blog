# atomic操作原子性

## 问题的提出

## 多线程的调试

https://blog.csdn.net/weixin_47410172/article/details/126161277

https://blog.csdn.net/hanxiaotongtong/article/details/107812976

https://springdoc.cn/spring-boot-actuator-startup/

原子类体系介绍

<img src="D:\picGo\images\image-20240531153205820.png" alt="image-20240531153205820" style="zoom:33%;" />

举一个实例说明 AtomicRefence 及其实际用法。

回过头看这段代码

```java
@Override
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
```

因为可能有多个线程去同时执行不同的步骤，

比如在执行spring.beans.instantiate的同时可能也会spring.data.repository.scanning。

这就告诉我们，在start的过程中，需要给出确定序列的id，并用BufferedStartupStep.compareAndSet去保证各执行步骤StartupStep的序列。



纳米精度

SpringApplicationRunListeners#doWithListeners方法去start各绑定的事件。



EventPublishingRunListener委托SimpleApplicationEventMulticaster执行#multicastEvent(SpringApplicationEvent事件)

![image-20240531170640380](D:\picGo\images\image-20240531170640380.png)







![image-20240531172417356](D:\picGo\images\image-20240531172417356.png)



自己的building

spring.beans.instantiate 361次

spring.data.repository.scanning

