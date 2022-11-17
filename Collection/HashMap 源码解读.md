# HashMap 源码解读

![img](https://s2.loli.net/2022/08/09/icjlPk86h5IKgWr.jpg)



## native

* native 关键字说明其修饰的方法是一个**原生态方法**，方法对应的实现不在当前文件，而是在用其他语言(C/C++) 实现的文件中

* Java 语言本身不能对操作系统底层进行访问和操作，但是可以通过 JNI 接口调用其他语言来实现对底层的访问

  > JNI 是 Java 本机接口 ( Java native Interface)，是一个本机编程接口，它是 Java 软件开发工具箱 SDK 的一部分。 JNI 允许 Java 代码使用其他语言编写的代码和代码库 



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

<img src="https://s2.loli.net/2022/08/09/4SD6vTdE3rJeUIF.png" alt="image-20220809205856831" style="zoom:50%;" />

**字符串**

![image-20220809210127717](https://s2.loli.net/2022/08/09/5xo4g7zeqZl9Ytp.png)

### 奇素数

31 不仅仅是符合 2^n-1 , 它是个奇素数（既是奇数，也是素数，也就是质数）

* 素数和其他数相乘的结果比其他方式更容易产生唯一性，减少哈希冲突
* 最终选择 31 是经过观测分布结果后的选择









## 计算 hash 值得到 Index

* 取余： index = k.hashCode() % entrys.length



% length   -> & (length   - 1)

### 为什么 length   - 1？

* 因为我们将数组长度设置为 2 的幂，这样减 1 的话二进制位中就全部为 1 了，这样进行与操作再转换为十进制正好是索引
* 另一个原因，即便我们不采用位运算的方式简化取模操作，数组的长度也应该设计为**素数(质数)**，**可以大大减少哈希冲突**，质数的定义是只能被自己和 1 整除





<img src="https://s2.loli.net/2022/08/09/lcdpfHBn9Z15DPg.png" alt="image-20220809211727671" style="zoom:50%;" />

> 取模方式导致我们 key 冲突的概率是非常大的
>
> 我们需要降低 Hash 冲突的概率，均匀存放每个下标的位置，可以直接根据下标位置定位到该元素，时间复杂度为 O(1)
>
> 那么，我们如何降低 Hash 冲突的概率的概率呢？答案就在源码中

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
* HashTable 安全，不允许存放  Key 为空









## HashMap 底层如何实现？

* 基于 ArrayList
  * List<Entry<K,V>> list
  * 优点：不需要考虑 hash 碰撞
  * 缺点：时间复杂度 O(n)
* jdk 1.7 ： 数组 + 链表
  * 数组初始容量：16
* jdk 1.8 ： 数组 + 链表 + 红黑树

HashMap 类似容器，使用 Entry 对象存放键值对

> 数组容量 > 64 且 链表长度 > 8 的时候，链表转化为红黑树 为什么是这个值呢，根据源码上的注释，这是根据泊松分布的规律得出来的
>
> 红黑树节点个数 < 6 ，转换为链表
>
> -> 为什么需要转换？
>
> * 红黑树节点包含更多信息，不仅需要保存节点的值，节点间的顺序关系，还有节点的颜色
> * 链表则相对简单



## HashMap  中 Key 为 null 存放什么位置？

HashMap 线程不安全、允许存放 Key 为空，放在数组 0 的位置





## HashMap 如何合理指定初始大小





## HashMap 为什么不适合遍历所有数据

HashMap  无序存放和散列的特点决定了，遍历所有 Key 的时候，需要将所有链表、红黑树都实现遍历，效率非常低



## HashMap 查询时间复杂度

* ArrayList： O(n)
* 数组 + 链表: 
  * 没有发生哈希冲突:  O(1)
  * 发生哈希冲突: O(1) + O(n)
* 数组 + 链表 + 红黑树 
  * 没有发生哈希冲突:  O(1)
  * 发生哈希冲突: O(1) + O(log n)



## HashMap 如何解决数组扩容问题







## LinkedHashMap

有序的 HashMap 集合 存放顺序 双向链表

原理：将每个 index 中的链表实现关联

效率比 HashMap 要低

LinkedHashMap 在 HashMap 的基础上维护元素添加顺序，使得**遍历的结果是遵从添加顺序的**

LinkedHashMap 基于双向链表实现，可以分为插入或者访问顺序两种，采用双向链表的形式保证有序，可以根据插入或者读取顺序

LinkedHashMap  是 HashMap 子类，但是内部还有一个双向链表维护键值对的顺序，每个键值对既位于哈希表中，也位于双向链表中。

LinkedHashMap  支持两种顺序：插入顺序、访问顺序

* 插入顺序：先添加的在前面，后添加的在后面。修改操作不影响顺序
* 访问顺序：执行 get/put 操作后，其对应的键值对会移动到链表末尾，所以末尾的是最近访问的，最开始的是最久没有被访问的



> LinkedHashMap 中有一个成员变量  accessOrder，默认为 false，如果设置为 true，就是访问顺序，false 就是插入顺序



## 缓存淘汰算法

Redis 缓存如果满的话，如果处理

* LRU : 最近少使用
* LFU : 最不经常使用
* ARC：自适应缓存替换
* FIFO： 先进先出
* MRU：最近最常使用



### LRU

* 方案1： 访问 key 的时候记录 count 值  -> 效率非常低
* 方案2： 基于 LinkedHashMap 有序集合实现
  * 原理：访问该 key 的时候就会将该 key 存放到链表最后的位置，链表最开头位置说明最近有可能少使用





