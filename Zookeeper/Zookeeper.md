# Zookeeper

## 为什么要有 Zookeeper

> 从设计模式角度来看：是一个基于**观察者模式**设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后**接受观察者的注册**。一旦这些数据的状态发生变化，**基于 Watch 机制**，Zookeeper 就负责通**知已经在 Zookeeper 上注册的那些观察者做出相应的反应**。
>
> Zookeeper = 文件系统 + 通知机制

* **分布式协调服务，可以用作注册中心和分布式锁**
* 基于**观察者模式**
* Zookeeper 用作注册中心，可以看作是 **文件系统 + 通知机制**。通知机制基于 （**Watch**）
* **支持集群服务 + 顺序一致性， 以 CP 闻名**，也就是强一致性。Leader 负责写，假如 Leader 挂了，重新选举 Leader 这段时间，服务是不可用的。但是官网给出测试，重新选举这段时间不会超过 200 ms







zxid: 高低三十二位

顺序一致性： 客户端发送的每一次请求到 Zookeeper 的时候都是有序的，在集群内也是有序的。

ZooKeeper `给每个请求都编了一个号`，比如第一次请求就叫 zxid-1，第二次请求就递增，叫 zxid-2……以此类推。

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgd3e7ec9de25a47be848acd5e0d966dc4tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="image.png" style="zoom:50%;" />

---



### Map<Master/Slave, IP> 的弊端

直接放到`Map<Master/Slave, IP>`里，请求进来后选择是 Master 还是 Slave，然后拿到 Ip，与之建立连接，进行操作等。

这个设计存在的问题：

* 基于内存的，没有持久化功能，挂掉以后就无法恢复
* 基于单 Jvm 的，也就是单机的，存在单点故障。集群部署的话， Jvm 内存中的数据如何同步？

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img85ebe68cf51448349b4c6d18400f722atplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="单机设计.png" style="zoom:50%;" />

解决以上问题的方法很容易想到

* 基于内存数据不安全，那么搞个持久化功能。写完内存后同步到磁盘一份
* 单点故障，集群部署。通过网络传输数据，收到网络请求的结点将数据持久化到磁盘并且装入内存。

这就是 Zookeeper 的原型

![分布式设计.png](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img31bf7d42fcf24844b9f1318fa5cce8f6tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)



## 使用 Zookeeper 的优势

* 很好地支持集群部署
* 具有很强地分布式协调能力（Watch 机制）。假如 Zookeeper 中地数据做了变更，这时候 Zookeeper 会主动通知其他监听这个数据的客户端，这份元数据有变更



## Eurka/Nacos 区别

ZooKeeper 是 CP，而 Eureka 是 AP，Nacos 既可以是 AP 又可以是 CP。

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgdc9fc872cf4241d5b1743daabc44d619tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="zk-ap-1.png" style="zoom:50%;" />

我们的 ZooKeeper 集群不是 Leader 负责写，写成功后不是同步到各个 Follower 从节点吗？那么问题来了，如果这时候 Leader 挂了，Follower 会进行选举，但是选举也需要时间的，<u>选举过程中如果进来了读写请求，那么是无法进行的</u>。所以会有部分流量的丢失，这就是所谓的 `CP 模型`，**用服务的可用性来换取数据的相对强一致性。**



那 ZooKeeper 的选主机制效率高吗？官方给了个压测的结果，**不会超过 200ms**。





## 特点

* **支持集群部署**。Zookeeper： 一个 领导者（Leader）， 多个 跟随者（Follower）组成的集群
* **高可靠性**，集群中只要有半数以上节点存活，Zookeeper 集群就能正常服务。Zookeeper  适合安装奇数台服务器
* **一致性**
  * 全局数据一致：每个 Server 保存一份相同的数据副本，Client 无论连接到哪个 Server 数据都是一致的。
  * **顺序一致性**。更新请求顺序执行，来自同一个 Client 的更新请求按其发送顺序依次执行

* 数据更新**原子性**，一次数据更新要么成功，要么失败。这里指的是一次请求在整个**分布式集群中具备原子性**，即要么整个集群中所有 ZooKeeper 节点都成功地处理了这个请求，要么就都没有处理，绝对不会出现集群中一部分 ZooKeeper 节点处理了这个请求，而另一部分节点没有处理的情况。
* **实时性**，一旦数据发生变化，那要及时通知给客户端。这个采取的是 Watcher 监听机制

---

* **Leader** 是`集群之首`，读写都可以提供，且写请求只能由 Leader 来完成，也负责将写请求的数据同步给各个节点。
* **Follower 是从节点的概念，仅提供读能力，且有资格参与选举**。
  * Follower 是分布式系统中常说的从节点，它只提供读请求的能力。<u>如果写请求到 Follower 上了，那么 Follower 会将此请求转发到 Leader，由 Leader 来完成写入，再同步给 Follower。</u>Follower 不只提供读能力，还额外<u>负责选举</u>，也就是比如 Leader 挂了的话，Follower 是有资格成为 Leader 的。
* **Observer也是从节点的概念，仅提供读能力，但是没有资格参与选举。**这是和 Follower 的差别所在。合理利用 Observer 可以<u>提供集群的读并发能力。</u>

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img1f356dfb12224281b365f6fdf4dc4d86tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="image.png" style="zoom:50%;" />



## 节点类型

- **持久节点**：即使客户端断开连接，那么此节点也一直存在。
- **临时节点**：如果客户端断开连接，此时它之前创建的临时节点就会自动消失、自动删除掉。所以说，如果客户端采取  ZooKeeper  的 watch 机制监听了这个临时节点，那么这个临时节点一旦被删除，客户端就会立即收到一个“临时节点已删除”的通知，这时候就可以处理一些自己的业务逻辑。

**持久节点就是持久化的，连接断开后数据也不会丢失；临时节点是基于内存的，连接断开后，节点就被删除了，数据也就随之丢失了**。



使用场景

- 首先来看持久节点。如果是做元数据存储，那持久节点是首选。
  - 比如 Kafka 创建了一个 topic，用 ZooKeeper 来管理元数据，topic 也是其中之一，那如果不用持久节点，用临时节点的话，客户端断开连接后 topic 就消失了？明显不对，所以如果是**做元数据存储那一定是持久节点。**
- 其次来看临时节点。**如果是做分布式协调和通知，那用临时节点**的比较多。
  - 比如：创建一个临时节点，然后客户端来监听这个临时节点的变化，如果我断开连接了，那么这时候临时节点会消失，此时监听这个临时节点的客户端会通过 Watcher 机制感知到变化。
- 最后看下临时顺序节点。**主要应用于在分布式锁的领域，** 
  - 在加锁的时候，会创建一个临时顺序节点，比如：`lock0000000000`，这时候其他客户端在尝试加锁的时候会继续自增编号，比如`lock0000000001`，且会注册 Watcher 监听上一个临时顺序节点，然后如果你客户端断开连接了，由于是临时节点，所以会自动销毁你加的这把锁，那么下一个编号为`lock0000000001`的客户端会收到通知，然后去尝试持有锁。

## 数据结构

Zookeeper 数据模型采用层次化的多叉树形结构，每个结点上都可以存储数据。

每个节点称为一个 ZNode，默认能够保存 1MB 数据，每个 ZNode 都可以通过其路径唯一标识。

![znode.png](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img4bc49ecaa2be4d59b8f94ccff1d9d26dtplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

```
#!/bin/bash

for host in 192.168.64.129 192.168.64.130 192.168.64.131
do
        case $1 in
        "start"){
                echo "------------ $host zookeeper -----------"
                ssh $host "source /etc/profile; /opt/module/apache-zookeeper-3.5.7/bin/zkServer.sh  start"
        };;
        "stop"){
                echo "------------ $host zookeeper -----------"
                ssh $host "source /etc/profile; /opt/module/apache-zookeeper-3.5.7/bin/zkServer.sh  stop"
        };;
        "status"){
                echo "------------ $host zookeeper -----------"
                ssh $host "source /etc/profile; /opt/module/apache-zookeeper-3.5.7/bin/zkServer.sh  status"
        };;
        esac
done
```



## Curator

Curator是Netflix公司开源的一套Zookeeper客户端框架。帮助我们在 Zookeeper 原生 API 基础上进行更好的封装

目前已经作为Apache的顶级项目出现，是最流行的Zookeeper客户端之一。



### 使用方法

1 导入依赖

![image-20220902164717960](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220902164717960.png)

2 连接 Zookeeper 客户端

```Java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3);
CuratorFramework zkClient = CuratorFrameworkFactory.builder()
    .connectString("192.168.64.129")
    .retryPolicy(retryPolicy)
    .build();
zkClient.start();
```

3 创建结点

![image-20220902180452721](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220902180452721.png)

![image-20220902180502140](Pic/image-20220902180502140.png)

4 删除结点

![image-20220902180539770](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220902180539770.png)

5 获取/更新结点

![image-20220902180604297](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220902180604297.png)

监听器

给某个结点注册子节点监听器。

注册了监听器之后，这个结点的子结点发生变化（增加、减少、更新等），可以自定义回调操作

![image-20220902180802180](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220902180802180.png)



## Leader 选举

---

- 什么是 Leader 选举？

  整个 ZooKeeper 集群只有一个 Leader，当 Leader 宕机后，集群不可对外提供写请求，相当于单（Leader）点故障，这时候 Follower 需要投票重新选择一个 Follower 作为 Leader。

- 选择哪个 Follower 作为 Leader 呢？

  先对比 epoch，epoch 大的优先被选择。epoch 一样的话就继续对比 zxid 的事务日志次数，事务日志次数大的代表数据最新，优先被选举；若事务日志次数也一样的话，那就选择一个 myid 最大的升级为 Leader。

- 那么什么是 epoch？什么是事务日志次数呢？

  epoch 和事务日志次数统称为 zxid：高 32 位是 Leader 的 epoch，从 1 开始，每次选出新的 Leader，epoch 加 1；低 32 位为该 epoch 内的序号，每次 epoch 变化，都将低 32 位的序号重置。这样保证了 zxid 的全局递增性。

- Leader 选主过程中涉及到哪些状态？

  `LOOKING`、`FOLLOWING`、`OBSERVING`、`LEADING`四种。

其实我觉得 ZooKeeper 有一个点的设计很巧妙，那就是 **zxid，巧妙采取高低 32 位用一个 long 类型字段代表两种含义，然后通过位运算高效率地进行运算**，好处在于不用逻辑啰嗦地维护两个字段，两个字段之间还是有强关联的，还需要保证这两个字段是原子操作。类似这种设计不光是 ZooKeeper 里有，我们 JDK 里的`读写锁`也是一个字段代表读锁和写锁两种锁

---





当 Leader 宕机后，Follower 要立即开始进行选主，选主的意思就是从 Follower 当中选择一个“最优秀”的出来，让它升级为 Leader。



我们 zxid 本身设计是一个自增的 long 类型字段，long 类型是 64 位的，我们可以把前 32 位作为**朝代次数**，后 32 位作为**事务日志id**，然后这前 32 位结合后 32 位统称为 zxid。

**zxid 是一个 long 类型的字段，前 32 位代表朝代，我们这里称为 epoch，后 32 位称为事务次数，也就是写请求次数。**

epoch 越大的就代表越新，因为每次选主都会自增 epoch，那先判断 epoch 不就好啦，epoch 越大的优先被选举；epoch 相同的，再对比 zxid 的后 32 位，也就是事务日志次数，值越大代表写的数据越多，那肯定数据越新；假设事务日志次数也一样，那就按照老套路对比 myid，找到 myid 最大的那个 Server 即可。

<img src="https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img434235aea32e4f568644437f4c05c521tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="持久化-4.png" style="zoom: 67%;" />

---



**选举状态**。它包含如下值：

- `LOOKING`：竞选状态，也就是说此状态下还没有 Leader 诞生，需要进行 Leader 选举。
- `FOLLOWING`：Follower 状态，对应我们之前介绍的 Follower 角色对应的状态，并且它自身是知道 Leader 是谁的。
- `OBSERVING`：Observer 状态，对应我们之前介绍的 Observer 角色对应的状态，并且它自身是知道 Leader 是谁的。
- `LEADING`：Leader 状态，对应我们之前介绍的 Leader 角色对应的状态。



> 服务器那么多，且服务器都是自己知道自己的数据，不知道其他服务器的数据（ zxid / myid ），那我们该怎么对比 zxid + myid 呢？

1. 如果接收到的 logicClock 大于自己的 logicClock，说明该服务器的选举轮次落后于其他服务器的选举轮次，那么先把自己的 logicalclock 更新为收到的，然后立即清空自己维护的票箱，接着对比 myid、zxid、epoch，看看收到的票据和自己的票据哪个更适合作为 Leader，最终再次将自己的投票通过网络广播出去。

   > *我们来举个例子：*
   >
   > 比如服务器 A 和服务器 B 两个节点，服务器 A 的 logicClock 为 2，服务器 B 的 logicClock 为 3，那么服务器 A 收到了服务器 B 的投票信息，这时候服务器 A 发现服务器 B 的轮次（logicClock）更大，那就代表服务器 A 是落后的，则会立即清空自己维护的票箱并把自己的 logicClock 改为服务器 B 的 logicClock，然后对比 myid、 zxid 和 epoch，看看哪个更适合成为 Leader，最后通过网络广播给其他机器。

2. 如果接收到的 logicClock 等于自己维护的 logicClock，那就对比二者的 vote_zxid，也就是对比被推荐服务器上所保存数据的最大 zxid。若收到的 vote_zxid 大于自己 vote_zxid，那就将自己票中的 vote_zxid 和 vote_id（被推荐服务器的 myid ）改为收到的票中的 vote_zxid 和 vote_id 并通过网络广播出去。另外将收到的票据和自己刚改后的 vote_id 放入自己的票箱。

   > *我们来举个例子：*
   >
   > 比如服务器 A 和服务器 B 两个节点，服务器 A 和服务器 B 的 logicClock 都为 2，但是服务器 A 的 vote_zxid 是 16，服务器 B 的 vote_zxid 是 20，那么服务器 A 收到了服务器 B 的投票信息，这时候服务器 A 发现服务器 B 的 vote_zxid 比自己的大，那就代表服务器 B 推荐的节点上的数据是最新的，则会将服务器 A 的 vote_zxid 从 16 改为 20，且修改服务器 A 的 vote_id 为服务器 B 的 vote_id，并通过网络广播给其他机器。

3. 如果接收到的 logicClock 等于自己维护的 logicClock 且二者的 vote_zxid 也一致，那就比较二者的 vote_id，也就是被推荐服务器的 myid。若接收到的投票的 vote_id 大于自己所选的 vote_id，那就将自己票中的 vote_id 改为接收到的票中的 vote_id 并通过网络广播出去。另外，将收到的票据和自己刚改后的 vote_id 放入自己的票箱。

   > 这个就无需举例了吧～

4. 如果接收到的 logicClock 小于自己的 logicClock，那么当前服务器直接忽略该投票，继续处理下一个投票。

这个选举投票流程我相信已经很清晰了，唯一有一点瑕疵就是：logicClock 啥时候自增呢？**每一轮投票就给 logicClock 加 1，这个后面我们分析源码的时候一目了然。**



**过半即可，也就是过半原则`(n+1/2)`，也就是超过半数以上的节点同意即可被选举为 Leader**。



##  ZAB协议的核心

ZAB协议的核心是，定义了如何处理那些会改变Zookeeper服务器数据状态的事务请求。即：

**所有的事务请求(会改变服务器数据状态的请求，如修改节点数据、删除节点等)必须由一个全局唯一的服务器来协调处理，该服务器被称为Leader，而剩余的其他服务器被称为Follower。Leader负责将一个客户端的事务请求转换成一个事务Proposal(提议)，并将该Proposal分发给集群中的所有Follower。然后Leader等待所有Follower的反馈结果，一旦有超过半数的Follower做出了正确的反馈后，Leader就会向所有的Follower再次发送Commit消息，要求将前一个Proposal提交。**




ZAB协议的两种模式：

1. 崩溃恢复

   当Zookeeper集群初始化时，或Leader故障宕机时，ZAB协议就会进入崩溃恢复模式，并选举出新的Leader。当新的Leader选举出来后，并且集群中已经有过半的节点与Leader完成了数据同步，ZAB协议就会退出崩溃恢复模式，转而进入消息广播模式。一个节点要想成为Leader，必须获得集群中过半节点的支持。

2. 消息广播

   ZAB的正常工作模式，如前文所说，Zookeeper设计成只允许唯一的一个Leader节点负责处理客户端的事务请求，当Leader接收到事务请求后，会生成相应的事务Proposal并发起一轮消息广播。如果集群中的非Leader节点(Follower或Observer)接收到了事务请求，会将请求转发给Leader处理。

   当Leader宕机，或者是集群中已经不存在超过半数的节点与Leader保持正常通信，那么集群就会进入崩溃恢复模式。



## Zookeeper 脑裂问题

https://juejin.cn/post/6844903895387340813

对于一个集群，想要提高这个集群的可用性，通常会采取多机房部署。

正常情况下，这个集群只有一个 Leader，而如果机房之间的网络断了，两个机房内的 zkServer 还是可以相互通信的。

如果不考虑过半机制，那么就会出现，每个机房内部都将选出一个 Leader。

这个问题，就相当于，原本一个集群，被分成了两个大脑，出现脑裂。如果过了一会，断了的网络突然通了，那么此时就会出现问题，两个集群刚刚都对外提供服务了，数据该怎么合并，数据冲突如何解决。

**实际上，Zookeeper 是不会出现脑裂问题的，原因就是过半选举机制。**



## Zookeeper 保证的是最终一致性

ZAB协议**认为只要是过半数节点写入成为**，数据就算写成功了，然后会告诉客户端A数据写入成功，如果这个时候客户端B恰好访问到还没同步最新数据的zookeeper节点，那么读到的数据就是不一致性的，因此zookeeper无法保证写数据的强一致性，只能保证最终一致性，而且可以保证同一客户端的顺序一致性。

* 如果节点之间存在网络延迟，而又要所有节点都同步数据才算成功，那么写的性能非常差
* 如果有一个节点挂了，无法同步数据，那么此时整个集群就无法提供写服务，无法保证可用性



## ZAB协议并不是Paxos算法的一个典型实现，两者联系

- 两者都存在一个类似于Leader进程的角色，由其负责协调多个Follower进程的运行
- Leader进程都会等待超过半数的Follower做出正确的反馈后，才会将一个提案进行提交
- 在ZAB协议中，每个Proposal中都包含一个epoch值，用来代表当前Leader周期，在Paxos算法中，同样类似标识——Ballot 在Paxos算法中，一个新选举产生的主进程会进行两个阶段的工作。一个读阶段，这个主进程会通过和所有其他进程进行通信的方式收集上一个主进程提出的提案，并将它们提交。一个写阶段，当前主进程开始提出它自己的提案。在Paxos算法设计的基础上，ZAB协议额外添加了一个同步阶段。在同步阶段之前，ZAB协议也存在一个和Paxos算法中的读阶段非常类似的过程，称之为发现（Discovery）阶段。在同步阶段，新的Leader会确保存在过半的Follower已经提交了之前Leader周期中的所有事务Proposal。这一阶段的引入，能够有效地保证Leader在新的周期中提出事务Proposal之前，所有进程已经完成之前所有事务Proposal的提交。一旦完成同步阶段后，那么ZAB就会执行和Paxos算法类似的写阶段。

总的来讲，ZAB协议和Paxos算法的本质区别在于，两者的设计目标不太一样。ZAB协议主要用于构建一个高可用的分布式数据主备系统，例如Zookeeper，而Paxos算法则是用于构建一个分布式的一致性状态机系统。



