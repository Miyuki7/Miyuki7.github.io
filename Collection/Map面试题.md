# Map面试题

## 哈希碰撞产生原因，如何解决

hashCode 值相等，但是对象内容不同

* jdk 1.7 ： 数组 + 链表
* jdk 1.8 ： 数组 + 链表 + 红黑树



## hashCode 的计算

* 整数: 整数值当作 hashCode, 比如 10 的 hashCode 就是 10
* 浮点数：将存储的二进制格式转为整数值 ( floatToIntBits(value) )
* Long 和 Double： <u>高 32 位</u> 和 <u>低 32 位</u> 做异或运算   ->   充分利用所有信息计算 hashCode，见下图
* 字符：字符的本质就是整数
* 字符串：直接见下图

**Long 和 Double**

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img4SD6vTdE3rJeUIF.png" alt="image-20220809205856831" style="zoom:50%;" />

**字符串**

![image-20220809210127717](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img5xo4g7zeqZl9Ytp.png)

### 奇素数

31 不仅仅是符合 2^n-1 , 它是个奇素数（既是奇数，也是素数，也就是质数）

* 素数和其他数相乘的结果比其他方式更容易产生唯一性，减少哈希冲突
* 最终选择 31 是经过观测分布结果后的选择





## 计算 hash 值得到 Index

* 取余： index = k.hashCode() % entrys.length



**% length   ->  & (length   - 1)**

### 为什么 length   - 1？

* 因为我们将**数组长度设置为 2 的幂，这样减 1 的话二进制位中就全部为 1 了**，这样进行与操作再转换为十进制正好是索引
* 另一个方面，即便我们不采用位运算的方式简化取模操作，数组的长度也应该设计为**素数(质数)**，**可以大大减少哈希冲突**，质数的定义是只能被自己和 1 整除

* **<u>这一点在我们的项目中也有使用，延时消息那部分。</u>**



<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imglcdpfHBn9Z15DPg.png" alt="image-20220809211727671" style="zoom:50%;" />

> 取模方式导致我们 key 冲突的概率是非常大的
>
> 我们需要降低 Hash 冲突的概率，均匀存放每个下标的位置，可以直接根据下标位置定位到该元素，时间复杂度为 O(1)
>
> 那么，我们**如何降低 Hash 冲突的概率的概率呢**？答案就在源码中

```Java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

```



## 为什么重写 equals 还要重写 hashcode

* Hashcode 方法： 方法被 native 修饰，底层采用 c 语言编写的，根据对象内存地址转换成整数类型
* Euqals 方法 ： 默认情况下 继承自 Object 类，采用 == 比较对象内存地址是否相等
* 如果两个对象的 Hashcode 值相等的情况下，对象的内容值不一定相等（**哈希碰撞**）；如果使用 Euqals 方法比较两个对象的内容值相等的情况下，则两个对象的 Hashcode 值相等
* 阿里巴巴开发手册规定了：只要重写 equals，就必须重写 hashCode，如果自定义对象作为 Map 的键，那么必须重写 equals 和 hashCode
* String 已经重写了上面两个方法，因此我们可以将 String 对象作为 key 使用
* **内存泄漏**： 如果 HashMap  使用 自定义对象 作为 key，举个极端的例子，new 多个不同对象，参数都是相同的内容，但是对象内存地址不同，但是没有重写 equals 和 hashCode，就有可能发生这样一个问题：无限死循环往里存，导致内存溢出。又涉及到一个强引用的问题



## HashMap  如何避免内存泄漏

HashMap  是否可以存放自定义对象作为 key ？

如果 HashMap  使用 自定义对象 作为 key，举个极端的例子，new 多个不同对象，参数都是相同的内容，但是对象内存地址不同，但是没有重写 equals 和 hashCode，就有可能发生这样一个问题：无限死循环往里存，导致内存溢出。又涉及到一个强引用的问题

* 解决方法：HashMap   对象使用自定义对象 作为 key，一定要重写 equals 和 hashCode。保证对象key 不会重复创建







## HashMap 与 HashTable 区别

* HashMap 线程不安全、允许存放 Key 为空，放在数组 0 的位置
* HashTable 线程安全，不允许存放  Key 为空
  * 在 HashTable 中，是直接在 put 和 get 方法上加上了 synchronized

## HashMap  中 Key 为 null 存放什么位置？

HashMap 线程不安全、允许存放 Key 为空，放在数组 0 的位置

ConcurrentHashMap，也不允许 key 和 value 为 空



## HashMap 底层如何实现？

* 基于 ArrayList
  * List<Entry<K,V>> list
  * 优点：不需要考虑 hash 碰撞
  * 缺点：时间复杂度 O(n)
* jdk 1.7 ： 数组 + 链表
  * 数组初始容量
* jdk 1.8 ： 数组 + 链表 + 红黑树

HashMap 类似容器，使用 Entry 对象存放键值对

> 数组容量 > 64 且 链表长度 > 8 的时候，链表转化为红黑树
>
> 红黑树节点个数 < 6 ，转换为链表
>
> -> 为什么需要转换？
>
> * 红黑树节点包含更多信息，不仅需要保存节点的值，节点间的顺序关系，还有节点的颜色
> * 链表则相对简单



## HashMap 为什么要用红黑树

* 在 jdk 1.8 版本之后， java 对 HashMap 做了改进。
* 在数组容量 > 64 且 链表长度 > 8 的时候，将后面的数据存在红黑树中，以加快检索速度，另一点考虑就是安全方面，防止有人恶意制造哈希冲突，将我们的 HashMap 退化为链表。
* 红黑树节点个数 < 6 ，转换为链表。
* 在聊一下红黑树的性质。红黑树本质上是一颗二叉搜索树，但是在二叉搜索树的基础上增加了着色和相关性的性质使得**红黑树更加平衡**，在相同情况下树高更低，查询效率更高，从而保证了红黑树的查找、插入、删除的时间复杂度为 0（logn）



## HashMap 为什么不适合遍历所有数据

HashMap  **无序存放和散列**的特点决定了，遍历所有 Key 的时候，需要将所有链表、红黑树都实现遍历，效率非常低

> * HashMap 特点： 无序存放 + 散列

## HashMap 查询时间复杂度

* ArrayList： O(n)
* 数组 + 链表: 
  * 没有发生哈希冲突:  O(1)
  * 发生哈希冲突: O(1) + O(n)
* 数组 + 链表 + 红黑树 
  * 没有发生哈希冲突:  O(1)
  * 发生哈希冲突: O(1) + O(log n)



## HashMap 如何解决数组扩容问题



## ConcurrentHashMap

https://juejin.cn/post/6961288808742518798

https://mvbbb.cn/concurrenthashmap-deepunderstanding/

https://tech.meituan.com/2016/06/24/java-hashmap.html

http://ifeve.com/java-concurrent-hashmap-1/

http://ifeve.com/java-concurrent-hashmap-2/

### 为什么使用ConcurrentHashMap

- HashMap在多线程中进行put方法有可能导致程序死循环，因为多线程可能会导致**HashMap形成环形链表**，(即链表的一个节点的next节点永不为null，就会产生死循环),会导致CPU的利用率接近100%，因此并发情况下不能使用HashMap。
- HashTable通过使用 **synchronized** 保证线程安全，但在线程竞争激烈的情况下效率低下。因为当一个线程访问 HashTable 的同步方法时，其他线程只能阻塞等待占用线程操作完毕。
- ConcurrentHashMap使用**分段锁**的思想，**对于不同的数据段使用不同的锁**，**可以支持多个线程同时访问不同的数据段，这样线程之间就不存在锁竞争，从而提高了并发效率**。

### 原理

`ConcurrentHashMap` 是 `HashMap` 线程安全的版本，其内部也和 HashMap 一样，采用了 数组 + 链表 + 红黑树的方式来实现。初始化也和 HashMap 一样，数组长度采用 2 的 N 次幂 

如何实现线程的安全性？最容易想到的就是加锁。

比如在 `HashTable` 中，是直接在 put 和 get 方法上加了 syn 锁。

但是这样做有缺点，syn 涉及偏向锁、轻量级锁、重量级锁，还有锁升级的过程，其中重量级锁还需要使用操作系统的 mutex 来实现，涉及到用户态内核态的切换。锁的粒度太大，会影响并发性能。

### JDK 1.7

#### Unsafe

Unsafe 类相当于 Java 语言中的后门类，提供了硬件级别的原子操作，所以在一些并发编程中被大量使用。但是对程序员而言，不是一个安全操作



#### 初始化容量

segment 是 16，但是每个 segment 内部还有一个数组，长度为 2 .因此初始化后的容量应当是 32







### JDK 1.8

### 初始化

```Java
public ConcurrentHashMap() {
    }
```

初始化的时候，如果不传参数 capacity ，默认构造函数什么都不做。有点懒加载的意思。

**建议我们会给出初始容量。 因为扩容比较耗时**。

在 JDK 1.7 ，给出容量 32 ，就是 32.

但是在 JDK 1.8 中，还会进行如下操作。目的是获取比 32 更大的那一个 2 的幂次方数

```Java
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```







### Jdk 1.8 改进

在 JDK 1.7 中，`ConcurrentHashMap` 由数组 + Segment + 分段锁实现。

内部分为一个个段数组，Segment 通过继承 ReentrantLock 来进行加锁，通过每次锁住一个 Segment 来降低锁的粒度，而且保证了每个 Segment 内部操作的线程安全性，从而实现全局线程安全。

![img](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img3dbec0c74fcd4a138548ce557cbc27f2tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

但是这么做的缺陷就是每次通过 hash 确认位置时需要 2 次才能定位到当前的 key 应该落在哪个槽

* 通过 hash 值和 段数组长度-1 进行位运算确认当前 key 属于哪个段，即确认其在 segments 数组的位置。

* 再次通过 hash 值和 table 数组（即 ConcurrentHashMap 底层存储数据的数组）长度 - 1进行位运算确认其所在桶。

为了进一步优化性能，在 jdk 1.8 版本中，对 ConcurrentHashMap 做了优化，取消了分段锁的涉及，取而代之的是通过 **cas 和 syn 关键字**实现优化，并且扩容的时候也利用了一种分而治之的思想来提升扩容效率。



---

#### JDK1.7与JDK1.8中ConcurrentHashMap的区别

其实可以看出JDK1.8版本的ConcurrentHashMap的数据结构已经接近 HashMap，

- 相对而言，ConcurrentHashMap只是增加了同步的操作来控制并发，从 JDK1.7版本的**ReentrantLock+Segment+HashEntry，到JDK1.8版本中synchronized+CAS+HashEntry+红黑树**。
- 保证**线程安全机制**：
  - JDK1.7采用 segment的分段锁机制实现线程安全，其中segment继承自 ReentrantLock。JDK1.8采用 CAS+Synchronized 保证线程安全。
  - 锁的粒度：原来是对需要进行数据操作的 Segment 加锁，现调整为对每个数组元素加锁（Node）。
- **链表转化为红黑树**:定位结点的 hash 算法简化会带来弊端 ,Hash冲突加剧,因此在链表节点数量大于8时，会将链表转化为红黑树进行存储。查询时间复杂度：从原来的遍历链表O(n)，变成遍历红黑树O(logN)。
- **JDK8中的扩容性能更高， 支持多线程同时扩容**， 实际上JDK7中也支持多线程扩容， 因为JDK7中的扩 容是针对每个Segment的， 所以多线程扩容性能没有JDK8高， 因为JDK8中对于任意一个线程都可以去帮助扩容
- **JDK8中的元素个数统计的实现也不一 样了**， JDK8中增加了 CounterCell 来帮助计数， 而JDK7中没有， JDK7中是 put 的时候每个 Segment 内部计数， 统计的时候是遍历每个 Segment 对象加锁统计。





### `sizeCtl`含义解释

> **注意：构造方法中，都涉及到一个变量`sizeCtl`，这个变量是一个非常重要的变量，而且具有非常丰富的含义，它的值不同，对应的含义也不一样，这里我们先对这个变量不同的值的含义做一下说明，后续源码分析过程中，进一步解释**
>
> `sizeCtl`为0，代表数组未初始化， 且数组的初始容量为16
>
> `sizeCtl`为正数，如果数组未初始化，那么其记录的是数组的初始容量，如果数组已经初始化，那么其记录的是数组的扩容阈值
>
> `sizeCtl`为-1，表示数组正在进行初始化
>
> `sizeCtl`小于0，并且不是-1，表示数组正在扩容， -(1+n)，表示此时有n个线程正在共同完成数组的扩容操作





### 为什么不允许 key 和 value 为 null

* 在并发编程中，**null 值容易引来歧义**，

* 假如先调用 get(key) 返回的结果是 null，那么我们<u>无法确认是因为当时这个 key 对应的 value 本身放的就是 null，还是说这个 key 值根本不存在</u>，这会引起歧义。

* 如果在非并发编程中，可以进一步通过调用 containsKey 方法来进行判断，但是并发编程中无法保证两个方法之间没有其他线程来修改 key 值，所以就直接禁止了 null 值的存在。

假设 ConcurrentHashMap 允许存放值为 null 的 value，这时有A、B两个线程，线程A调用`ConcurrentHashMap.get(key)`方法，返回为 null ，我们不知道这个 null 是没有映射的 null ，还是存的值就是 null 。

假设此时，返回为 null 的真实情况是没有找到对应的 key。那么，我们可以用 `ConcurrentHashMap.containsKey(key)`来验证我们的假设是否成立，我们期望的结果是返回 false 。

但是在我们调用 `ConcurrentHashMap.get(key)`方法之后，`containsKey`方法之前，线程B执行了`ConcurrentHashMap.put(key, null)`的操作。那么我们调用`containsKey`方法返回的就是 true 了，这就与我们的假设的真实情况不符合了，这就有了二义性。

### 如何保证线程安全性

在 ConcurrentHashMap 中，采用了大量的分而治之的思想来降低锁的粒度，提升并发性能。其源码中**大量使用了 cas 操作来保证安全性**，而不是和 HashTable 一样，不论什么方法，直接简单粗暴的使用 synchronized关键字来实现。

#### 保证初始化安全



#### put 操作保证数组元素可见性

> 不是对整个数组加锁，是对一个桶位加锁



当向ConcurrentHashMap中put一 个key ,value时，

1. 首先根据 key 计算对应的数组下标 i， 如果该位置没有元素， 则通过自旋的方法去向该位置赋值。

2. 如果该位置有元素， 则 synchronized 会加锁

3. 加锁成功之后， 在判断该元素的类型a. 如果是链表节点则进行添加节点到链表中b. 如果是红黑树则添加节点到红黑树

4. 添加成功后，判断是否需要进行树化

5. addCount，这个方法的意思是ConcurrentHashMap的元素个数加1， 但是这个操作也是需要并发安全的，并且元素个数加1成功后，会继续判断是否要进行扩容， 如果需要，则会进行扩容，所以这个方法很重要。

6 .  同时一个线程在put时如果发现当前ConcurrentHashMap正在进行扩容则会去帮助扩容。

counterCell

### 扩容







### ConcurrentHashMap的get方法是否要加锁，为什么？

因为 Node 的元**素 value 和指针 next 是用 volatile 修饰的**，在多线程环境下线程A修改节点的 value 或者新增节点的时候是对线程B可见的。从 Hash 表中读取数据，与扩容也不冲突

这也是它比其他并发集合比如 Hashtable、用 Collections.synchronizedMap()包装的 HashMap 效率高的原因之一。



### ConcurrentHashMap 的并发度是什么？

并发度可以理解为程序运行时能够同时更新 ConccurentHashMap且不产生锁竞争的最大线程数。在JDK1.7中，实际上就是ConcurrentHashMap中的**分段锁**个数，即Segment[]的数组长度，默认是16，这个值可以在构造函数中设置。

如果自己设置了并发度，ConcurrentHashMap 会使用大于等于该值的最小的2的幂指数作为实际并发度，也就是比如你设置的值是17，那么实际并发度是32。

如果并发度设置的过小，**会带来严重的锁竞争问题**；**如果并发度设置的过大，原本位于同一个Segment内的访问会扩散到不同的Segment中，CPU cache命中率会下降，从而引起程序性能下降**。

在JDK1.8中，已经摒弃了Segment的概念，选择了Node数组+链表+红黑树结构，**并发度大小依赖于数组的大小**。

### ConcurrentHashMap 迭代器是强一致性还是弱一致性？

与 HashMap 迭代器是强一致性不同，ConcurrentHashMap 迭代器是弱一致性。

**ConcurrentHashMap 的迭代器创建后，就会按照哈希表结构遍历每个元素，但在遍历过程中，内部元素可能会发生变化，如果变化发生在已遍历过的部分，迭代器就不会反映出来，而如果变化发生在未遍历过的部分，迭代器就会发现并反映出来，这就是弱一致性。**

ConcurrentHashMap的弱一致性主要是为了**提升效率，是一致性与效率之间的一种权衡**。







