## Spring-Cloud-Gateway源码系列学习

版本 v2.2.6.RELEASE



### RouteDefinitionLocator 与 RouteLocator整体设计

#### RouteDefinitionLocator 与 RouteLocator设计图：

![img](https://static.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/01.png)

tip：RouteDefinitionLocator的职责是将各种配置源的配置数据转化成RouteDefinition，而RouteLocator的职责是把RouteDefinition转化成Route，关于RouteDefinition与Route等基础组件的介绍可以参考本系列#Spring-Cloud-Gateway基础组件学习，在RoutePredicateHandlerMapping类里面定义了一个routeLocator，可以看出在路由选择是根据RouteLocator#getRoutes来获取Flux<Route>，然后尝试匹配

```
private final RouteLocator routeLocator;
```

------

##### RouteDefinitionLocator相关类简介

- **RouteDefinitionLocator**：各种数据源的RouteDefinitionLocator顶级接口，里面只有一个方法，获得RouteDefinition流，Flux是Reactor的类

  ```
  Flux<RouteDefinition> getRouteDefinitions()
  ```

- **PropertiesRouteDefinitionLocator**：从配置文件(YAML / Properties等)读取并解析成Flux<RouteDefinition>，解析部分使用了Spring-boot的配置功能（站在巨人的肩膀上），即标注了@ConfigurationProperties的GatewayProperties

- **RouteDefinitionRepository**：继承 RouteDefinitionLocator、RouteDefinitionWriter，是个空接口，代表从存储器(内存 / Redis / MySQL 等)读取并解析成Flux<RouteDefinition>，但在v2.2.6.RELEASE版本只发现InMemoryRouteDefinitionRepository(内存的RouteDefinitionRepository实现)一个实现

  - **RouteDefinitionWriter**：是个接口，里面只有两个方法，保存和删除，需要子类实现，即需要

    ```java
    //保存一个RouteDefinition，Flux、Mono都是Publisher<T>，而Flux表示0-多个元素的流，而Mono表示最多一个元素的流
    Mono<Void> save(Mono<RouteDefinition> route);
    
    //根据routeId删除RouteDefinition
    Mono<Void> delete(Mono<String> routeId);
    ```

- **DiscoveryClientRouteDefinitionLocator**：从注册中心(Eureka / Consul / Zookeeper / Etcd 等)读取并解析成Flux<RouteDefinition>

- **CompositeRouteDefinitionLocator**：组合上面的配置文件、存储器、注册中心的RouteDefinitionLocator 的实现，委派设计模式，为 RouteDefinitionRouteLocator 提供统一入口。

  ```java
  //RouteDefinitionLocator的数据流
  private final Flux<RouteDefinitionLocator> delegates;
  
  //routeId生成策略
  private final IdGenerator idGenerator;
  ```

------

##### RouteLocator相关类简介

- **RouteDefinitionRouteLocator**：