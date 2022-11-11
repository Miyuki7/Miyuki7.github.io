##  spring事务

spring支持两种事务管理方式，一是声明式事务管理（xml或者@Transactional注解），一种是编程式事务管理（TransactionTemplate`或者TransactionManager）。

![image-20221110200716385](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221110200716385.png)

### 七种事务传播行为

``` java
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    ......
}
```

* **`TransactionDefinition.PROPAGATION_REQUIRED`**:如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

  * **在外围方法未开启事务的情况下`Propagation.REQUIRED`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。**
  * **在外围方法开启事务的情况下`Propagation.REQUIRED`修饰的内部方法会加入到外围方法的事务中，所有`Propagation.REQUIRED`修饰的内部方法和外围方法均属于同一事务，只要一个方法回滚，整个事务均回滚。**

* **`TransactionDefinition.PROPAGATION_REQUIRES_NEW`**:创建一个新的事务，如果当前存在事务，则把当前事务挂起。也就是说不管外部方法是否开启事务，`Propagation.REQUIRES_NEW`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。

  * **在外围方法未开启事务的情况下`Propagation.REQUIRES_NEW`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰（和Propagation.REQUIRES一个效果）。**
  * **在外围方法开启事务的情况下`Propagation.REQUIRES_NEW`修饰的内部方法依然会单独开启独立事务，且与外部方法事务也独立，内部方法之间、内部方法和外部方法事务均相互独立，互不干扰。**

* **`TransactionDefinition.PROPAGATION_NESTED`**:如果当前存在事务，就在嵌套事务内执行；如果当前没有事务，就执行与`TransactionDefinition.PROPAGATION_REQUIRED`类似的操作。

  * 在外部方法开启事务的情况下,在内部开启一个新的事务，作为嵌套事务存在。
  * 如果外部方法无事务，则单独开启一个事务，与 `PROPAGATION_REQUIRED` 类似。
  * 外围方法开启事务的情况下`Propagation.NESTED`修饰的内部方法属于外部事务的子事务，外围主事务回滚，子事务一定回滚，而内部子事务可以单独回滚而不影响外围主事务和其他子事务

* REQUIRED,REQUIRES_NEW,NESTED 异同

  * **NESTED 和 REQUIRED 修饰的内部方法都属于外围方法事务，如果外围方法抛出异常，这两种方法的事务都会被回滚。但是 REQUIRED 是加入外围方法事务，所以和外围事务同属于一个事务，一旦 REQUIRED 事务抛出异常被回滚，外围方法事务也将被回滚。而 NESTED 是外围方法的子事务，有单独的保存点，所以 NESTED 方法抛出异常被回滚，不会影响到外围方法的事务。**

  * **NESTED 和 REQUIRES_NEW 都可以做到内部方法事务回滚而不影响外围方法事务。但是因为 NESTED 是嵌套事务，所以外围方法回滚之后，作为外围方法事务的子事务也会被回滚。而 REQUIRES_NEW 是通过开启新的事务实现的，内部事务和外围事务是两个事务，外围事务回滚不会影响内部事务。**

* **`TransactionDefinition.PROPAGATION_MANDATORY`**：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

* **`TransactionDefinition.PROPAGATION_SUPPORTS`**: 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。

* **`TransactionDefinition.PROPAGATION_NOT_SUPPORTED`**: 以非事务方式运行，如果当前存在事务，则把当前事务挂起。

* **`TransactionDefinition.PROPAGATION_NEVER`**: 以非事务方式运行，如果当前存在事务，则抛出异常。

  ![640](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img640.png)

### 事务隔离级别

```java
public interface TransactionDefinition {
    ......
    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED = 2;
    int ISOLATION_REPEATABLE_READ = 4;
    int ISOLATION_SERIALIZABLE = 8;
    ......
}
```

- **`TransactionDefinition.ISOLATION_DEFAULT`** :数据库默认的隔离级别，MySQL 默认采用的 `REPEATABLE_READ` 隔离级别 Oracle 默认采用的 `READ_COMMITTED` 隔离级别.
- **`TransactionDefinition.ISOLATION_READ_UNCOMMITTED`** :最低的隔离级别，使用这个隔离级别很少，因为它允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **`TransactionDefinition.ISOLATION_READ_COMMITTED`** : 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **`TransactionDefinition.ISOLATION_REPEATABLE_READ`** : 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **`TransactionDefinition.ISOLATION_SERIALIZABLE`** : 最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

### 事务超时属性

指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 `TransactionDefinition` 中以 int 的值来表示超时时间，其单位是秒，默认值为-1。

### 事务只读属性

对于只有读取数据查询的事务，可以指定事务类型为 readonly，只读事务不涉及数据的修改，数据库会提供一些优化手段，适合用在有多条数据库查询操作的方法中。

* 使用场景：执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询 SQL 必须保证整体的读一致性，否则，在前条 SQL 查询之后，后条 SQL 查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持

### 事务回滚规则

默认情况下，事务只有遇到运行期异常（`RuntimeException` 的子类）时才会回滚，`Error` 也会导致事务回滚，但是，在遇到检查型（Checked）异常时不会回滚。

* 可以自己定义要回滚的异常

  ```java
  @Transactional(rollbackFor= MyException.class)
  ```

### @Transactional`的作用范围和原理

* 方法 ：推荐将注解使用于方法上，该注解只能应用到 public 方法上，否则不生效。

* 类 ：如果这个注解使用在类上的话，表明该注解对该类中所有的 public 方法都生效。

* 不推荐在接口上使用。

* `Transactional` 的工作机制是基于 AOP 实现的，AOP 又是使用动态代理实现的。如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理，如果目标对象没有实现了接口,会使用 CGLIB 动态代理。如果一个类或者一个类中的 public 方法上被标注`@Transactional` 注解的话，Spring 容器就会在启动的时候为其创建一个代理类实际调用的是，`TransactionInterceptor` 类中的 `invoke()`方法。

*  `@Transactional` 的使用注意事项总结
- `@Transactional` 注解只有作用到 public 方法上事务才生效，不推荐在接口上使用；
  - 避免同一个类中调用 `@Transactional` 注解的方法，这样会导致事务失效；
  - 正确的设置 `@Transactional` 的 `rollbackFor` 和 `propagation` 属性，否则事务可能会回滚失败;
  - 被 `@Transactional` 注解的方法所在的类必须被 Spring 管理，否则不生效；
  - 底层使用的数据库必须支持事务机制，否则不生效

### 自调用失效问题

若同一类中的其他没有 `@Transactional` 注解的方法内部调用有 `@Transactional` 注解的方法，有`@Transactional` 注解的方法的事务会失效。

这是由于`Spring AOP`代理的原因造成的，因为只有当 `@Transactional` 注解的方法在类以外被调用的时候，Spring 事务管理才生效。

解决方法

* 使用`AopContext.currentProxy()`方法来获取到当前类的代理对象，调用代理对象的B方法，也是可以完成事务的，还可以通过`@Autowired`将自身的代理对象引入，不过写法没有本质的改变，都需要对内部调用的代码进行针对性的修改，不太完美。

  ``` java
   public void testAsync() {
          System.out.println(Thread.currentThread().getName());
          System.out.println("async1");
        ((AsyncMethod)AopContext.currentProxy()).testAsnc2();
      }
  	@Async
      public void testAsnc2() {
          System.out.println(Thread.currentThread().getName());
          System.out.println("async2");
      }
  ```

  

* 使用 AspectJ 取代 Spring AOP 代理，使用AspectJ编译器会将切面织入到目标class。


参考文章：[太难了~面试官让我结合案例讲讲自己对Spring事务传播行为的理解。 (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486668&idx=2&sn=0381e8c836442f46bdc5367170234abb&chksm=cea24307f9d5ca11c96943b3ccfa1fc70dc97dd87d9c540388581f8fe6d805ff548dff5f6b5b&token=1776990505&lang=zh_CN#rd)

[Spring 事务总结 | JavaGuide](https://javaguide.cn/system-design/framework/spring/spring-transaction.html#事务属性详解)