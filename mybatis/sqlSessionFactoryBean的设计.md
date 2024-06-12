# SqlSessionFactoryBean的设计

## spring与mybatis的整合

在mybatis-spring的整合中，mybatis将核心启动组件SqlSessionFactory扩展了一个SqlSessionFactoryBean

这个bean是一个FactoryBean，统一了spring ioc的行为。

这个bean还去实现了InitializingBean，并在该bean初始化后确保SqlSessionFactory的正确性。





而在spring中，只需配置SqlSessionFactoryBean来引入SqlSessionFactory，即在spring项目启动时能加载并创建SqlSessionFactory对象，然后注册到spring的IOC容器中

## 源码分析

```java
public class SqlSessionFactoryBean
    implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {
   // 事务管理，mybatis接入spring的一个重要原因也是可以直接使用spring提供的事务管理
  private TransactionFactory transactionFactory;
    
  //1.getObject方法，在创建SqlSessionFactory时会用这个方法去获取bean
    @Override
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();//完成SqlSessionFactory的赋值
    }

    return this.sqlSessionFactory;
  }
  //2.afterPropertiesSet方法，在bean初始化后会被spring调用
  @Override
  public void afterPropertiesSet() throws Exception {
    this.sqlSessionFactory = buildSqlSessionFactory();
  }
    
}

```

如上，SqlSessionFactoryBean实现了spring的FactoryBean<SqlSessionFactory>接口，spring在去创建“SqlSessionFactory”这个bean时，会从工厂方法（getObject）去获取。

而在getObject方法中，主动调用了InitializingBean的afterPropertiesSet方法，在这个方法里关键的一行代码是buildSqlSessionFactory。下面看一下它做了什么

```java
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    // 配置类
    final Configuration targetConfiguration;

    // 解析mybatisConfig.xml文件，将相关配置信息保存到targetConfiguration
    XMLConfigBuilder xmlConfigBuilder = null;
    
    ...

    if (xmlConfigBuilder != null) {
      try {
        xmlConfigBuilder.parse();//解析mybatisConfig.xml到targetConfiguration中
        LOGGER.debug(() -> "Parsed configuration file: '" + this.configLocation + "'");
      }
      
      ...
      
    }
    // 为sqlSessionFactory绑定事务管理器和数据源，
    // 这样sqlSessionFactory在创建sqlSession的时候可以通过该事务管理器获取jdbc连接，从而执行SQL
    targetConfiguration.setEnvironment(new Environment(this.environment,
        this.transactionFactory == null ? new SpringManagedTransactionFactory() : this.transactionFactory,
        this.dataSource));

    // 解析*mapper.xml
    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        try {
          // 解析mapper.xml文件，并注册到configuration对象的mapperRegistry
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              targetConfiguration, mapperLocation.toString(), targetConfiguration.getSqlFragments());
          xmlMapperBuilder.parse();
        } 
        
      }
    }
    ...
    // 将Configuration对象实例作为参数，
    // 调用sqlSessionFactoryBuilder创建sqlSessionFactory对象实例
    return this.sqlSessionFactoryBuilder.build(targetConfiguration);
}

```

> 如上，buildSqlSessionFactory方法会对mybatis-config文件解析，存到targetConfiguration中
>
> 并设置事务管理器和dataSource，用于获取jdbc连接，执行sql
>
> 根据对应路径，去解析*mapper.xml，并解析到targetConfiguration的mapperRegistry中
>
> 最终用sqlSessionFactoryBuilder去拿这个Configuration对象(targetConfiguration)，去创建一个SqlSessionFactory

总结一下就是

- 1.mybatis-config.xml和*mapper.xml解析到targetConfiguration中’
- 2.用sqlSessionFactoryBuilder去拿这个Configuration对象(targetConfiguration)，去创建一个DefaultSqlSessionFactory



**那么sqlSessionFactoryBuilder.build发生了什么**

```java
package org.apache.ibatis.session.defaults;//mybatis的builder，和spring无关  
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }

public class DefaultSqlSessionFactory implements SqlSessionFactory
    private final Configuration configuration;
  public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
```

如上，会调用mybatis的builder去创建一个DefaultSqlSessionFactory（**和spring无关**），这时的数据如下：

![image-20240608202334375](D:\picGo\images\image-20240608202334375.png)

可以看到SqlSessionFactory正是一个default，并且configuration保存着mybatis-config.xml和*mapper.xml解析的数据，并environment里面存储了配置的事务管理器和数据源。这时的SqlSessionFactory就很完美，可以通过它去取得jdbc连接，获取session，执行sql了。

