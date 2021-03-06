来源：jianshu.com/p/e674ee68fd3f



# **一、为什么要用锁？**

锁-是为了解决并发操作引起的脏读、数据不一致的问题。



# **二、锁实现的基本原理**

## **2.1、volatile**

Java编程语言允许线程访问共享变量， 为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了[volatile](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247483916&idx=1&sn=89daf388da0d6fe40dc54e9a4018baeb&chksm=eb53873adc240e2cf55400f3261228d08fc943c4f196566e995681549c47630b70ac01b75031&scene=21#wechat_redirect)，在某些情况下比锁要更加方便。

volatile在多处理器开发中保证了共享变量的“ 可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。

![img](assets\20190509100421.jpg)

结论：如果volatile变量修饰符使用恰当的话，它比synchronized的使用和执行成本更低，因为它不会引起线程上下文的切换和调度。

## **2.2、synchronized**

synchronized通过锁机制实现同步。



先来看下利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁。

具体表现为以下3种形式。

- 对于普通同步方法，锁是当前实例对象。
- 对于静态同步方法，锁是当前类的Class对象。
- 对于同步方法块，锁是Synchonized括号里配置的对象。



当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。

### **2.2.1 synchronized实现原理**

synchronized是基于Monitor来实现同步的。

Monitor从两个方面来支持线程之间的同步：

- 互斥执行
- 协作

1、Java 使用对象锁 ( 使用 synchronized 获得对象锁 ) 保证工作在共享的数据集上的线程互斥执行。[面试常考：Synchronized 有几种用法？](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247487913&idx=2&sn=318f4cbbde0669bd8c62af3501d2c915&chksm=eb53949fdc241d89be6b54cb3db4e7367a3ac4894c2911ac3fa236708dfb0360355660fb0ca4&scene=21#wechat_redirect)

2、使用 notify/notifyAll/wait 方法来协同不同线程之间的工作。

3、Class和Object都关联了一个Monitor。

![img](assets\20190509100431.jpg)

Monitor 的工作机理

- 线程进入同步方法中。
- 为了继续执行临界区代码，线程必须获取 Monitor 锁。如果获取锁成功，将成为该监视者对象的拥有者。任一时刻内，监视者对象只属于一个活动线程（The Owner）
- 拥有监视者对象的线程可以调用 wait() 进入等待集合（Wait Set），同时释放监视锁，进入等待状态。
- 其他线程调用 notify() / notifyAll() 接口唤醒等待集合中的线程，这些等待的线程需要重新获取监视锁后才能执行 wait() 之后的代码。
- 同步方法执行完毕了，线程退出临界区，并释放监视锁。

参考文档：https://www.ibm.com/developerworks/cn/java/j-lo-synchronized

### **2.2.2 synchronized具体实现**

1、同步代码块采用monitorenter、monitorexit指令显式的实现。

2、同步方法则使用ACC_SYNCHRONIZED标记符隐式的实现。

通过实例来看看具体实现：

![img](assets\20190509100441.jpg)

javap编译后的字节码如下：

![img](assets\20190509100448.jpg)

**monitorenter**

每一个对象都有一个monitor，一个monitor只能被一个线程拥有。当一个线程执行到monitorenter指令时会尝试获取相应对象的monitor，获取规则如下：

- 如果monitor的进入数为0，则该线程可以进入monitor，并将monitor进入数设置为1，该线程即为monitor的拥有者。
- 如果当前线程已经拥有该monitor，只是重新进入，则进入monitor的进入数加1，所以synchronized关键字实现的锁是可重入的锁。
- 如果monitor已被其他线程拥有，则当前线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor。

**monitorexit**

只有拥有相应对象的monitor的线程才能执行monitorexit指令。每执行一次该指令monitor进入数减1，当进入数为0时当前线程释放monitor，此时其他阻塞的线程将可以尝试获取该monitor。

### **2.2.3 锁存放的位置**

锁标记存放在Java对象头的Mark Word中。

![img](assets\20190509100458.jpg)

Java对象头长度

![img](assets\20190509100504.jpg)

32位JVM Mark Word 结构

![img](assets\20190509100510.jpg)

32位JVM Mark Word 状态变化

![img](assets\20190509100516.jpg)

64位JVM Mark Word 结构

### **2.2.3 synchronized的锁优化**

JavaSE1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。[Java 虚拟机对锁优化所做的努力](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247486696&idx=2&sn=444dc545ac8b183095839dfda146b155&chksm=eb5389dedc2400c8c3db9f4879a7e99040e9e3947de0d3f3d564494ba2efb48361f707ee9329&scene=21#wechat_redirect)。

在JavaSE1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。

锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

#### **偏向锁：**

无锁竞争的情况下为了减少锁竞争的资源开销，引入偏向锁。

![img](assets\20190509100523.jpg)

#### **轻量级锁：**

轻量级锁所适应的场景是线程交替执行同步块的情况。

![img](assets\20190509100530.jpg)

**锁粗化**（Lock Coarsening）：也就是减少不必要的紧连在一起的unlock，lock操作，将多个连续的锁扩展成一个范围更大的锁。

**锁消除**（Lock Elimination）：锁削除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行削除。

**适应性自旋**（Adaptive Spinning）：自适应意味着自旋的时间不再固定了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将允许自旋等待持续相对更长的时间，比如100个循环。另一方面，如果对于某个锁，自旋很少成功获得过，那在以后要获取这个锁时将可能省略掉自旋过程，以避免浪费处理器资源。

### **2.2.4 锁的优缺点对比**

![img](assets\20190509100537.jpg)

## **2.3、CAS**

CAS，在Java并发应用中通常指CompareAndSwap或CompareAndSet，即比较并交换。[深入理解CAS算法原理](http://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247483728&idx=1&sn=3d734dc972a244891406cfbc443eabed&chksm=eb538466dc240d7033b889665b579a490266b8c8f1e7a08da35f67ca484dad19503e8b230e05&scene=21#wechat_redirect)。

1、CAS是一个原子操作，它比较一个内存位置的值并且只有相等时修改这个内存位置的值为新的值，保证了新的值总是基于最新的信息计算的，如果有其他线程在这期间修改了这个值则CAS失败。CAS返回是否成功或者内存位置原来的值用于判断是否CAS成功。

2、JVM中的CAS操作是利用了处理器提供的CMPXCHG指令实现的。

优点：

- 竞争不大的时候系统开销小。

缺点：

- 循环时间长开销大。
- ABA问题。
- 只能保证一个共享变量的原子操作。

# 

# **三、Java中的锁实现**



## **3.1、队列同步器（AQS）**

队列同步器AbstractQueuedSynchronizer（以下简称同步器），是用来构建锁或者其他同步组件的基础框架。

### 3.1.1、它使用了一个int成员变量表示同步状态。

![img](assets\20190509100550.jpg)

### 3.1.2、通过内置的FIFO双向队列来完成获取锁线程的排队工作。

- 同步器包含两个节点类型的应用，一个指向头节点，一个指向尾节点，未获取到锁的线程会创建节点线程安全（compareAndSetTail）的加入队列尾部。同步队列遵循FIFO，首节点是获取同步状态成功的节点。

![img](assets\20190509100557.jpg)

- 未获取到锁的线程将创建一个节点，设置到尾节点。如下图所示：

![img](assets\20190509100603.jpg)

- 首节点的线程在释放锁时，将会唤醒后继节点。而后继节点将会在获取锁成功时将自己设置为首节点。如下图所示：

![img](assets\20190509100609.jpg)

### 3.1.3、独占式/共享式锁获取

独占式：有且只有一个线程能获取到锁，如：ReentrantLock；

共享式：可以多个线程同时获取到锁，如：CountDownLatch；

#### **独占式**

- 每个节点自旋观察自己的前一节点是不是Header节点，如果是，就去尝试获取锁。

![img](assets\20190509100617.jpg)

- 独占式锁获取流程：

![img](assets\20190509100624.jpg)

#### 共享式：

- 共享式与独占式的区别：

![img](assets\20190509100630.jpg)

- 共享锁获取流程：

![img](assets\20190509100637.jpg)

# 

# **四、锁的使用用例**



## **4.1、ConcurrentHashMap的实现原理及使用**

![img](assets\20190509100643.jpg)

ConcurrentHashMap类图

![img](assets\20190509100650.jpg)

ConcurrentHashMap数据结构

结论：ConcurrentHashMap使用的锁分段技术。首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。