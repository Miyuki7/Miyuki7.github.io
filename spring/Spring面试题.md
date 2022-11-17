# Spring

## 简单介绍一下 Spring，优缺点

* Spring 是重量级企业开发框架 **EJB** 的替代品，为企业级 Java 开发提供了一种相对简单的方法。通过 **依赖注入** 和 **面向切面编程**，用简单的 Java 对象实现了 EJB 的功能
* 虽然 Spring 的**组件代码是轻量级**的，但它的**配置是重量级**的，需要大量的 XML 配置
* 为此， 
  * Spring 2.5 引入了基于注解的组件扫描。消除了大量针对应用程序自身组件的显示 XML 配置。
  * Spring 3.0 引入了基于 Java 的配置，可以代替 XML
* 组件扫描减少了配置量，但是Spring 还是需要不少配置。比如 Servlet 和 过滤器，需要在 web.xml 和 Servlet 初始化代码里进行显式配置
* 除此之外，相关库的依赖以及不同库版本之间的冲突，会消耗我们大部分时间和精力



## 为什么要有 SpringBoot

简化 Spring 开发，减少配置文件，达到开箱即用的效果

**自动装配可以说是 Spring Boot 的核心**

引入 starter 之后，我们通过少量注解和一些简单的配置就能使用第三方组件提供的功能了。

## SpringBoot 的主要优点

* SpringBoot 使得开发 Spring 的应用程序更加容易。
  * 不需要编写大量样板代码、XML 配置 和 注释
  * SpringBoot 提供了嵌入式 HTTP 服务器，比如 Tomcat 和 Jetty，可以轻松地开发和测试 Web 应用程序，普通运行 Java 程序地方式就可以运行基于 SpringBoot 的 Web 项目 
    * 补充一下，SpringBoot 中 Tomcat 的入口，在 onRefresh 方法中 createWebserver 中 onStart 方法
  * 提供了多种插件，可以使用内置工具比如 Maven 开发和测试 SpringBoot 应用程序
  * SpringBoot 提供了一系列 “默认配置”，开发者可以根据需要修改，减少开发工作。
  * Spring 引导应用程序可以很容易地与 Spring 生态系统集成，比如 SpringData、Spring Security 等



## Spring MVC

![image-20220924203551617](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220924203551617.png)

![image-20220924204207567](Pic/image-20220924204207567.png)

## Spring 中的设计模式 *







## 什么是 SpringBoot Starters

SpringBoot Starters 是一系列依赖关系的集合，因为它的存在，项目的依赖之间的关系变得更加简单

比如我们开发 Web 应用程序，我们需要使用 Spring MVC， Tomcat 和 Jackson 这样的库，这些依赖要一个一个手动添加，但是有了 Starters ，我们只需要添加 web 对应的 starter 一个依赖就可以了。

[maven](https://so.csdn.net/so/search?q=maven&spm=1001.2101.3001.7020)的 install 可以将项目本身编译并打包到本地仓库，这样其他项目引用本项目的jar包时不用去私服上下载jar包，直接从本地就可以拿到刚刚编译打包好的项目的jar包，很灵活，避免每次都需要重新往私服发布jar包的痛苦；

## @SpringBootApplication 注解

@SpringBootApplication 大概可以看为 @Configuration, @EnableAutoConfiguration, @ComponentScan 注解的集合

* @Configuration： 允许上下文中注册额外的 bean 或导入其他配置类
* @EnableAutoConfiguration: 启动 SpringBoot 的自动配置机制
* @ComponentScan： 扫描被 @Component 注解的 bean，默认会扫描该类所在包下的所有类
  * @ComponentScan主要就是定义**扫描的路径**从中找出标识了**需要装配**的类自动装配到spring的bean容器中
  * **自定扫描路径下边带有@Controller，@Service，@Repository，@Component注解加入spring容器**
  * 这也就是为什么启动类的位置在 dao/controller 这些包的上级了，扫描下层所有包



## SpringBoot 自动装配 *

> Spring Boot 通过`@EnableAutoConfiguration`开启自动装配，通过 SpringFactoriesLoader 最终加载`META-INF/spring.factories`中的自动配置类实现自动装配，自动配置类其实就是通过`@Conditional`按需加载的配置类，想要其生效必须引入`spring-boot-starter-xxx`包实现起步依赖



**自动装配可以说是 Spring Boot 的核心**

引入 starter 之后，我们通过少量注解和一些简单的配置就能使用第三方组件提供的功能了。

> 我们使用一个公共 starter 的时候，只需要将相应的依赖添加到 Maven 的配置文件中即可，免去了自己引用很多依赖类，并且 SpringBoot 会进行类的自动装配。
>
> 那么 SpringBoot 如何知道要实例化哪些类，并进行自动配置呢？
>
> * 首先，SpringBoot 在启动时会去依赖的 starter 包中 寻找 `resources/META-INF/spring.factories` 文件，然后根据文件配置的 jar 包，去扫描项目所依赖的 jar 包。类似于 Java 的 SPI 机制
> * 第二步，根据 `spring.factories` 配置加载 AutoConfigure 类
> * 最后，根据 `@Conditional` 注解的条件，进行自动配置，并将 Bean 注入 Spring Context 上下文当中
> * 我们也可以使用`@ImportAutoConfiguration({MyServiceAutoConfiguration.class})` 指定自动配置哪些类。

**Spring 是如何实现自动装配的**

![image-20220903174608935](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903174608935.png)

* 首先在 Spring 的启动类上有一个核心注解： `@SpringBootApplication`。我们知道这个注解大体上可以看成 `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
* `@EnableAutoConfiguration` 是自动装配的核心注解。这个注解的主要功能，就是通过其上的 `@Import` 注解引入了一个关键类，`AutoConfigurationImportSelector`
* 在这个关键类 `AutoConfigurationImportSelector` 中，调用了一个方法，`selectImports`，用于获取所有符合条件的，类的全限定名，这些类最终被加载到 IOC 容器中
* 在进一步深入，`selectImports` 方法中主要干了两件事
  * 首先，if 判断自动装配的开关是否打开，不是重点
  * 其次，调用了一个关键方法 `getAutoConfigurationEntry()` ，这个方法是负责加载自动配置类的。
  *  `getAutoConfigurationEntry()` 方法，最终是通过 SpringFactoriesLoader 从`META-INF/spring.factories` 文件加载自动配置类的

![image-20220903172353000](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903172353000.png)



**“`spring.factories`中这么多配置，每次启动都要全部加载么？”。**

条件注解 @ConditionalOnXXX 注解起到了筛选作用，只有满足全部条件的类才会生效，加载到 IOC 容器中





---



编写自己的 springboot-starter

https://www.cnblogs.com/yuansc/p/9088212.html

* 创建工程，导入 spring-boot-starter 依赖
* 在配置类上添加注解 `@Configuration`，创建自己的Bean，可以使用注解 `@ConditionOnXXX` 配置加载的条件
* 在 `resources/META-INF/spring.factories`文件中，给出配置文件路径



[基于ImportBeanDefinitionRegistrar和FactoryBean动态注入Bean到Spring容器中](https://segmentfault.com/a/1190000039821699)

## 常用注解

https://javaguide.cn/system-design/framework/spring/spring-common-annotations.html#_0-%E5%89%8D%E8%A8%80



**Spring Bean 相关**

* `@Autowired`: 自动导入对象到类中，被注入进的类被 Spring 容器管理
* `@Component`: 可以**标注任意类为 Spring 组件**，如果一个 Bean 不知道属于哪个层，可以使用这个注解
* `@Repository`： 标注类为持久层，DAO 层，主要用于数据库相关操作
* `@Service`： 标注类为服务层，主要涉及一些复杂的逻辑
* `@Controller`： 标注类为控制层，主要用于接收用户请求并调用 Service 层返回数据给前端页面
* `@RestController`： 这个注解是 @Controller 和 @ResponseBody 的融合。表示这个 bean 是 REST风格的控制器，**将函数的返回值 直接 填入 HTTP 响应体中**



**处理 HTTP 请求相关：**

* `@GetMapping`
* `@PostMapping`
* `@PutMapping`
* `@DeleteMapping`



**前后端传值**

* `@RequestParm` 以及 `@PathVariable`: 前者用于获取查询参数，后者用于获取路径参数
* `@RequestBody`: 用于读取 Request 请求的 body 部分，并且 Content-Type 为 application/json 格式的数据，接收到数据之后会自动将数据绑定到 Java 对象上去。系统会使用 HttpMessageConverter 或者自定义的 HttpMessageConverter 将请求的 body 中的 json 字符串转换为 java 对象













## SpringBoot 常用的两种配置文件

`application.properties`

`application.yml`

后者更加直观清晰，有层次感



## 读取配置文件的方法

![image-20220903145750494](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903145750494.png)

### @Value 读取简单的配置信息

在变量上加上 @Value 注解，不推荐

![image-20220903145758647](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903145758647.png)

### @ConfigurationProperties 读取并与 bean 绑定

![image-20220903145807402](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903145807402.png)

### 通过 @ConfigurationProperties 读取并校验

![image-20220903150043849](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903150043849.png)

![image-20220903150033297](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903150033297.png)



### @PropertySouce 读取指定的 properties 文件

![image-20220903150158242](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903150158242.png)





## 配置文件的优先级

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221114234708335.png" alt="image-20221114234708335" style="zoom:50%;" />

优先级 1：当前 jar 包所在目录下的子目录 config 文件夹中下的子目录中
优先级 2：当前 jar 包所在目录下的子目录 config 文件夹中
优先级 3：当前 jar 包所在目录同级目录下
优先级 4：类路径（resource资源文件或者java目录下）下的子目录 config 文件夹中
优先级 5：类路径（resource资源文件或者java目录下）下的配置文件（默认创建的配置文件位置）



## 实现定时任务



定时任务上的 `@Schedule` 注解 + 启动类上的 `@EnableScheduling` 注解

![image-20220903150944631](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903150944631.png)

## @Component 注解 和 @Bean 注解的区别

* @Component 作用在类上，表示被注解的类是一个 Bean，@Bean 作用在方法上，表示方法的返回值是 Bean
* @Bean 注解自定义性更强，假如使用第三方库需要配置到 Spring 容器中，那么只能使用 Bean 注解



## 单例 Bean 的线程安全问题

大部分时候我们创建的 Bean，比如 Service，Dao 等等，都是无状态的，是线程安全的。

当我们需要在 Bean 中定义可变的成员变量，可以使用 ThreadLoacl



## 什么是 IOC？

简单的说，IOC 是反转控制。

* 第一点，IOC 的目的是：

  * **接口的具体实现**与**执行的任务**之间解耦

  * 关注于设计上的最终目标，而不是具体实现

  * 更加形象一些的描述是，1983 年 IOC 提出时候引入的一个概念叫好莱坞原则，“**好莱坞原则**” -> "不要打电话给我，我会打电话给你"。程序中也是如此，**我们的依赖和数据是由其他模块来进行提供，而非我们自己读取**。

* 第二点，IOC 的实现方式主要有，**依赖查找和依赖注入**。
  * 其中，依赖查找主动获取依赖，实现相对繁琐，代码会侵入业务逻辑，通常需要依赖容器或者标准 API 实现
  * 依赖注入这种实现方式呢是被动提供依赖，实现相对便利，具有低侵入性，无需依赖特定容器和 API。
* 第三点，按照 IOC 的定义，很多方面都是 IOC
  * **Spring 框架 是 IOC 容器的一种实现**。
  * 我们常说的 **JavaBeans 是 IOC 容器的一种实现**，**Servlet 的容器也是 IOC 的实现**，因为 Servlet 可以通过 依赖或者 反向地通过 JNDI 的方式得到一些外部的资源，包括 DataSource 和 EJB 的组件
* 第四点，按照反转控制来说呢，我们最近常说的 **Reactive 响应式编程，还有消息实际上也是 IOC 的一种体现**，因为消息实际上是一个被动的，我们传统的调用链路是一个主动拉的模式，那么 IOC 实际上是一种推的模式，那么我们说这种**消息、事件、以及反向的观察者模式的扩展**都属于 IOC

## 依赖注入总结分析*





## 依赖查找和依赖注入的区别

IOC 的实现方式主要有，**依赖查找和依赖注入**。

* 其中，依赖查找主动获取依赖，实现相对繁琐，代码会侵入业务逻辑，通常需要依赖容器或者标准 API 实现（比如 Servlet 的 API ，比如 JNDI 的 API）
* 依赖注入这种实现方式呢是被动提供依赖，实现相对遍历，具有低侵入性，无需依赖特定容器和 API。



## Spring 作为 IOC 容器有什么优势*

* 首先，Spring 完成了传统的、典型的 IOC 管理，比如依赖查找和依赖注入
* Spring 的 AOP 抽象做的非常好
* 事务抽象
* 事件机制的扩展（EventObject, EventListener），凡是事件监听，就标记 EventListener 进行扩展
* 提供了几种 SPI 的扩展方式，SpringFactories,自动装配的时候经常用到
* 强大的第三方整合，减少中间环节的需要。比如，Spring Data，封装了关系数据库常用的东西，CRUD，通过契约的方式来动态生成一些中间代码，帮助我们调用一些数据库的基本操作。统一 API ，不关注数据来源。
* 易测试性。 spring-test 模块提供的集成测试以及单元测试，用途广泛。



## 什么是 Spring IOC 容器

我们知道，IOC 容器的两种实现方式，依赖查找和依赖注入。

Spring 框架是 IOC 容器的一种实现，DI 是它实现的原则，那么，IOC 容器也就是，依赖注入的容器。

实际上 IOC 容器还有一个依赖查找的实现方式。



## 事务

####  事务传播行为

**事务传播行为是为了解决业务层方法之间互相调用的事务问题**。



## BeanFactory 与 FactoryBean

> FactoryBean通常是用来创建比较复杂的Bean，一般的bean使用XML配置即可，但若是一个Bean的创建过程中涉及到很多其他Bean 和 复杂的逻辑，使用XML配置比较复杂的话，这时可以考虑使用FactoryBean。



* BeanFactory 是 IOC 的底层容器。是一个工厂类，用于**管理Bean**的一个工厂，在Spring中，所有Bean都是由BeanFactory（也就是IOC容器）来进行管理的。

* FactoryBean 是一个工厂Bean，是**创建 Bean** 的一种方式，用户可以通过实现该接口定制实例化Bean的逻辑。帮助实现复杂的初始化逻辑

### 谁才是真正的 IOC 容器

*BeanFactory 和 ApplicationContext 谁才是 IOC 的容器？*

* **BeanFactory 是底层的 IOC 容器，提供了一个框架和基本的特性**，**ApplicationContext** 在这个基础上提供了更多企业级特性的功能，是 BeanFactory 的超集

* AbstractRefreshableApplicationContext 集成 AbstratcApplicationContext，在其中维**护了一个 DefaultListableBeanFactory 的引用**（不是继承关系）

* **ApplicationContext 是 BeanFactory  的子接口**

  * 提供了 AOP 特征资源的整合

  * 消息资源的处理（用于国际化）

  * 事件发布







下面是课程笔记

---



## 重新认识 IOC

* **接口的具体实现**与**执行的任务**之间解耦
* 关注于设计上的最终目标，而不是具体实现
* “好莱坞原则”，不要来找我们，我们来找你。“我们”，需要的东西或者资源，“你”，我们的系统或者模块。



### IOC 发展简介

编程的风格、编程的原则

软件架构会相对复杂

* 1983 年，“**好莱坞原则**” -> "不要打电话给我，我会打电话给你"。程序中也是如此，**我们的依赖和数据是由其他模块来进行提供，而非我们自己读取**。
  * 不严谨的说，个人看法，这一点有点像最近流行的 Reactive 响应式编程，同样都是反向的，不是主动拉取，是一个推的方式
* **2004年**发表的一篇论文中，IOC 和 DI 被整合在一起（在此之前就被提出），是我们理解的最终来源。



### IOC 主要实现策略

**依赖查找**和**依赖注入** 两个大类

* 依赖注入是一种 IOC 的主要实现策略
  * 构造器注入
  * 接口注入
  * 参数输入
  * set 注入
* 上下文依赖查询，Java Bean， beanContext
* 模板方法设计模式， JDBC 模板
* 策略模式

### IOC 容器的职责

通用职责

* 依赖处理
  * 依赖查找
    * 根据名称、类型查找
    * 涉及到类型转换
  * 依赖注入
* 生命周期管理
  * 容器（启动，停止，暂停，回滚）
  * 托管的资源 （Java Beans 或其他资源）
* 配置
  * 容器
  * 外部化的配置
  * 托管的资源

### IOC 容器的实现

* Java SE
  * Java Beans，Beans 的管理
  * Java ServiceLoader SPI
* Java EE
  * EJB，企业级 Java Bean
  * Servlet
* 开源
  * Spring 框架



#### 传统IOC 容器的实现

* Java Beans 作为 IOC 的容器
* 特性：
  * 依赖查找
  * 生命周期管理
  * 资源管理
  * 持久化
  * 事件
  * 配置元信息

#### 什么才是 Java Bean

简单的理解就是我们的 POJO

很大特性，有 set / get 方法，可写可读方法，字段称为属性

一个特性， BeanInfo，元信息。自省的方式，`Introspector.getBeanInfo`

`beanInfo.getPropertyDescriptions`

每个对象的 getClass 方法会被当作一个属性，因此要设定 stopClass



### 轻量级 IOC 容器





### 依赖查找 VS 依赖注入

| 类型     | 依赖获取 | 实现便利性 | 代码侵入性   | API 依赖性     | 可读性 |
| -------- | -------- | ---------- | ------------ | -------------- | ------ |
| 依赖查找 | 主动获取 | 相对繁琐   | 侵入业务逻辑 | 依赖容器 API   | 良好   |
| 依赖注入 | 被动提供 | 相对便利   | 低侵入性     | 不依赖容器 API | 一般   |



### 构造器注入 VS Setter 注入

https://blog.csdn.net/sdx1237/article/details/59173172



---



### 依赖查找

* 根据 Bean 名称查找
  * 实时查找 （`BeanFactory.getBean(name)`）
  * 延时查找（`ObjectFactory.getObject()`）
* 根据 Bean 类型进行查找
  * 单个 Bean 对象（`BeanFactory.getBean(User.class)`）
  * 集合 Bean 对象 (`ListableBeanFactory.getBeansOfType`)
* 根据 Bean 名称 + 类型查找
* 根据 Java 注解查找 (`ListableBeanFactory.getBeansWithAnnotation`)
  * 单个 Bean 对象
  * 集合 Bean 对象



### ObjectFactory/BeanFactory/Factory Bean



### 依赖注入

* 根据 Bean 名称注入
* 根据 Bean 类型注入
  * 单个 Bean 对象
  * 集合 Bean 对象
* 注入容器内建 Bean 对象
* 注入非 Bean 对象
* 注入类型
  * 实时注入
  * 延迟注入





### 依赖查找和注入的对象来自于哪里？

Sring IOC 依赖来源

* 自定义 Bean （User）
* 容器内建 Bean 对象（Environment）
* 容器内建依赖 （BeanFactory）



### Spring IOC 配置元信息

* Bean 的定义配置
  * 基于 XML 文件
  * 基于 Properties 文件
  * 基于 Java 注解
* IOC 容器配置
  * 基于 XML 文件
  * 基于 Java 注解
* 外部化属性配置
  * 基于 Java 注解

### 谁才是真正的 IOC 容器

*BeanFactory 和 ApplicationContext 谁才是 IOC 的容器？*

* **BeanFactory 是底层的 IOC 容器，提供了一个框架和基本的特性**，**ApplicationContext** 在这个基础上提供了更多企业级特性的功能，是 BeanFactory 的超集

* AbstractRefreshableApplicationContext 集成 AbstratcApplicationContext，在其中维**护了一个 DefaultListableBeanFactory 的引用**（不是继承关系）

* **ApplicationContext 是 BeanFactory  的子接口**

  * 提供了 AOP 特征资源的整合

  * 消息资源的处理（用于国际化）

  * 事件发布



### IOC 容器生命周期

* 启动
* 运行
* 停止





## Spring Bean

### 定义 Spring Bean

* BeanDefinition 是 Spring Framwork 中定义 Bean 的配置元信息接口，包含：
  * Bean 的类名
  * Bean 行为配置元素，如作用域、自动绑定的模式、生命周期回调等
  * 其他 Bean 引用，又可称为合作者或者依赖
  * 配置设置，比如 Bean 属性



| 属性                     | 说明                                          |
| ------------------------ | --------------------------------------------- |
| Class                    | Bean 全类名，必须是具体类，不能用抽象类或接口 |
| Name                     | Bean 的名称 或者 ID                           |
| Scope                    | Bean 的作用域 （singleton、prototype等）      |
| Constructor arguments    | Bean 构造器参数                               |
| Properties               | Bean 属性设置                                 |
| Autowiring mode          | Bean 自动绑定模式                             |
| Lazy initialization mode | Bean 延迟初始化模式                           |
| Initialization method    | Bean 初始化回调方法名称                       |
| Destruction method       | Bean 销毁回调方法名称                         |



### BeanDefinition 构建

* 通过 BeanDefinitionBuilder
* 通过 AbstractBeanDefinition 以及派生类



### 命名 Spring Bean

Bean 的名称

每个 Bean 拥有一个或多个标识符 （identifiers），这些标识符在 **Bean 所在的容器**必须是唯一的。通常，一个 Bean 仅有一个标识符，如果需要额外的，可考虑使用别名（Alias）来扩充。

在基于 XML 的配置元信息中，开发人员可用 **id 或者 name** 属性来规定 Bean 的标识符。

通常 Bean 的标识符由字母组成，允许出现特殊字符。如果要想引入 Bean 的别名的话，可在 name 属性使用半角逗号或分号来间隔

Bean 的 id 或 name 并非必须指定，若为空的话，容器会为 Bean 自动生成一个 唯一 的名称。

Bean 的命名尽管没有限制，不过官方建议采用驼峰的方式，更适合 Java 的命名约定



### BeanDefinition 注册

* XML 配置元信息
* Java 注解配置元信息
  * @Bean
  * @Component
  * @Import
* Java API 配置元信息
  * 命名方式
    * `registry.registerBeanDefinition(beanName,beanDefinitionBuilder.getBeanDefinition)` 注册 BeanDefinition
  * 非命名方式
  * 配置类方式





### 如何将 BeanDefinition 注册到 IOC 容器

 @Bean

* 通过 @Component
* 通过 @Import

```Java
applicationContext.register(Config.class)

applicationContext.refresh()
```



### 实例化 Bean

* 常规方式
  * 通过构造器
  * 通过静态工厂方法
  * 通过 Bean 工厂方法
  * 通过 FactoryBean
* 特殊方式
  * 通过 ServiceLoaderFactoryBean
  * 通过 AutowireCapableBeanFactory
  * 通过 BeanDefinitionRegistry





----

## AOP

面试： AOP 概述

> AOP 面向切面编程。原有的 Java OOP 存在一些局限，包括静态化语言和侵入性扩展。静态化语言是指的是，类结构一旦定义，很难修改。侵入性扩展指的是，只能通过继承和组合来组织新的类结构。
>
> OOP 关注的核心是对象，AOP 关注的核心是切面。
>
> AOP 可以在不修改功能代码本身的前提下，通过运行时的动态代理技术，对已有的代码逻辑增强。
>
> 此外，AOP 可以实现组件化、可插拔的功能扩展，通过简单的配置，就可以将功能增强到指定的切入点。











### AOP 引入

Java OOP 存在哪些局限

* 静态化语言，类结构一旦定义，不容易被修改
* 侵入性扩展：通过继承和组合组织新的类结构





### AOP 概述

* AOP 是 OOP 的补充
  * 将重复的代码逻辑抽取为一个切面，通过在运行时动态代理组合进原有的对象。实现预期效果
* AOP 关注的是核心切面
* AOP 也是 Spring IOC 的补充
  * 如果没有 AOP， IOC 本身也可以是 Spring 非常强大的特性
  * 只不过 AOP 可以在 IOC 容器中，针对需要的 Bean 去增强原有的功能



#### AOP 核心工作

使得原有业务与扩展逻辑之间，解耦



Proxy 代理对象 = Target 目标对象 + Advice 通知

Aspect 切面 = PointCut 切点 + Advice 通知



### 通知的类型

* Before 前置通知
* After 后置通知
* AfterReturning 返回通知
  * 目标对象的方法调用完成，在返回结果值之后触发
* AfterThrowing 异常通知
  * 目标对象的方法运行中抛出 / 触发异常后触发
* Around 环绕通知
  * 编程式控制目标对象的方法调用。环绕通知是所有通知类型中可操作范围最大的一种，因为它可以直接拿到目标对象，以及要执行的方法，所以环绕通知可以任意的在目标对象的方法调用前后搞事，甚至不调用目标对象的方法。



### AOP 使用场景

* 业务日志
* 权限校验
* 事务控制



### AOP 失效场景

* 代理对象调用自身方法的时候
* 代理对象在后置处理器还没有初始化的时候，提前创建了



### Spring AOP 和 AspectJ AOP 有什么区别

* Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。前者基于代理，后者基于字节码操作
* Spring AOP 已经集成了 AspectJ，AspectJ 功能更强大，Spring AOP 相对来说更简单
* 如果我们的切面比较少，两者性能差异不大。切面太多，最好选择 AspectJ



### 多个切面的执行顺序

使用 @Order 注解，值越小，优先级越高



