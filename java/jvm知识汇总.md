[TOC]



作者：说出你的愿望吧丷<br />
链接：https://juejin.im/post/5e1505d0f265da5d5d744050<br />
来源：掘金<br />
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。<br />

## 一、JVM基本介绍

![img](images/jvm知识汇总/16f8ab42da5a81cd)

### 1.1 **方法区** 

是用于存放类似于元数据信息方面的数据的，比如类信息，常量，静态变量，编译后代码···等。类加载器将 .class 文件搬过来就是先丢到这一块上

### 1.2 堆

 主要放了一些存储的数据，比如对象实例，数组···等，它和方法区都同属于 **线程共享区域** 。也就是说它们都是 **线程不安全** 的

### 1.3 栈
这是我们的代码运行空间。我们编写的每一个方法都会放到 **栈** 里面运行。

###  1.4 程序计数器
 主要就是完成一个加载工作，类似于一个指针一样的，指向下一行我们需要执行的代码。和栈一样，都是 **线程独享** 的，就是说每一个线程都会有自己对应的一块区域而不会存在并发和多线程的问题。

### 小结
方法区，堆都为线程共享区域，有线程安全问题，
栈和本地方法栈和计数器都是独享区域，不存在线程安全问题，
而 JVM 的调优主要就是围绕堆，栈两大块进行

## 二、运行时数据区

### 2.1 本地方法栈和程序计数器

比如说我们现在点开Thread类的源码，会看到它的start0方法带有一个native关键字修饰，而且不存在方法体，这种用native修饰的方法就是本地方法，这是使用C来实现的，然后一般这些方法都会放到一个叫做本地方法栈的区域。

程序计数器其实就是一个指针，它指向了我们程序中下一句需要执行的指令，它也是内存区域中唯一一个不会出现OutOfMemoryError的区域，而且占用内存空间小到基本可以忽略不计。这个内存仅代表当前线程所执行的字节码的行号指示器，字节码解析器通过改变这个计数器的值选取下一条需要执行的字节码指令。

如果执行的是native方法，那这个指针就不工作了。

### 2.2 方法区

方法区主要的作用技术存放类的元数据信息，常量和静态变量···等。当它存储的信息过大时，会在无法满足内存分配时报错。

### 2.3 虚拟机栈和虚拟机堆

一句话便是：栈管运行，堆管存储。则虚拟机栈负责运行代码，而虚拟机堆负责存储数据。

#### 2.3.1 虚拟机栈的概念

它是Java方法执行的内存模型。里面会对局部变量，动态链表，方法出口，栈的操作（入栈和出栈）进行存储，且线程独享。同时如果我们听到局部变量表，那也是在说虚拟机栈

#### 2.3.2 虚拟机栈存在的异常

如果线程请求的栈的深度大于虚拟机栈的最大深度，就会报 **StackOverflowError** （这种错误经常出现在递归中）。Java虚拟机也可以动态扩展，但随着扩展会不断地申请内存，当无法申请足够内存时就会报错 **OutOfMemoryError**。

#### 2.3.3 虚拟机栈的生命周期

对于栈来说，不存在垃圾回收。只要程序运行结束，栈的空间自然就会释放了。栈的生命周期和所处的线程是一致的。

这里补充一句：8种基本类型的变量+对象的引用变量+实例方法都是在栈里面分配内存。

#### 2.3.4 虚拟机堆的概念

JVM内存会划分为堆内存和非堆内存，堆内存中也会划分为**年轻代**和**老年代**，而非堆内存则为**永久代**。年轻代又会分为**Eden**和**Survivor**区。Survivor也会分为**FromPlace**和**ToPlace**，toPlace的survivor区域是空的。Eden，FromPlace和ToPlace的默认占比为 **8:1:1**。当然这个东西其实也可以通过一个 -XX:+UsePSAdaptiveSurvivorSizePolicy 参数来根据生成对象的速率动态调整

堆内存中存放的是对象，垃圾收集就是收集这些对象然后交给GC算法进行回收。非堆内存其实我们已经说过了，就是方法区。在1.8中已经移除永久代，替代品是一个元空间(MetaSpace)，最大区别是metaSpace是不存在于JVM中的，它使用的是本地内存。并有两个参数

```
MetaspaceSize：初始化元空间大小，控制发生GC
MaxMetaspaceSize：限制元空间大小上限，防止占用过多物理内存。
复制代码
```

移除的原因可以大致了解一下：融合HotSpot JVM和JRockit VM而做出的改变，因为JRockit是没有永久代的，不过这也间接性地解决了永久代的OOM问题

#### 2.3.5 Eden年轻代的介绍

当我们new一个对象后，会先放到Eden划分出来的一块作为存储空间的内存，但是我们知道对堆内存是线程共享的，所以有可能会出现两个对象共用一个内存的情况。这里JVM的处理是每个线程都会预先申请好一块连续的内存空间并规定了对象存放的位置，而如果空间不足会再申请多块内存空间。这个操作我们会称作TLAB，有兴趣可以了解一下。

当Eden空间满了之后，会触发一个叫做Minor GC（就是一个发生在年轻代的GC）的操作，存活下来的对象移动到Survivor0区。Survivor0区满后触发 Minor GC，就会将存活对象移动到Survivor1区，此时还会把from和to两个指针交换，这样保证了一段时间内总有一个survivor区为空且to所指向的survivor区为空。经过多次的 Minor GC后仍然存活的对象（**这里的存活判断是15次，对应到虚拟机参数为 -XX:TargetSurvivorRatio 。为什么是15，因为HotSpot会在对象投中的标记字段里记录年龄，分配到的空间仅有4位，所以最多只能记录到15**）会移动到老年代。老年代是存储长期存活的对象的，占满时就会触发我们最常听说的Full GC，期间会停止所有线程等待GC的完成。所以对于响应要求高的应用应该尽量去减少发生Full GC从而避免响应超时的问题。

而且当老年区执行了full gc之后仍然无法进行对象保存的操作，就会产生OOM，这时候就是虚拟机中的堆内存不足，原因可能会是堆内存设置的大小过小，这个可以通过参数-Xms、-Xms来调整。也可能是代码中创建的对象大且多，而且它们一直在被引用从而长时间垃圾收集无法收集它们。

![img](images/jvm知识汇总/16fa201c39ac86ad)

#### 2.3.6 如何判断一个对象需要被干掉

两个基础的计算方法:

1. 引用计数器计算
   给对象添加一个引用计数器，每次引用这个对象时计数器加一，引用失效时减一，计数器等于0时就是不会再次使用的。不过这个方法有一种情况就是出现对象的<font color="red">循环引用时GC没法回收</font>。
2. 可达性分析计算
   类似于二叉树的实现，将一系列的GC ROOTS作为起始的存活对象集，从这个节点往下搜索，搜索所走过的路径成为引用链，把能被该集合引用到的对象加入到集合中。搜索当一个对象到GC Roots没有使用任何引用链时，则说明该对象是不可用的。主流的商用程序语言，例如Java，C#等都是靠这招去判定对象是否存活的。
   这种方法的优点是能够解决循环引用的问题，可它的实现需要耗费大量资源和时间，也需要GC（它的分析过程引用关系不能发生变化，所以需要停止所有进程）



## 三、垃圾回收算法

常用的有标记清除，复制，标记整理和分代收集算法

#### 3.1 标记清除算法

标记清除算法就是分为“标记”和“清除”两个阶段。标记出所有需要回收的对象，标记结束后统一回收。这个套路很简单，也存在不足，后续的算法都是根据这个基础来加以改进的。

其实它就是把已死亡的对象标记为空闲内存，然后记录在一个空闲列表中，当我们需要new一个对象时，内存管理模块会从空闲列表中寻找空闲的内存来分给新的对象。

不足的方面就是标记和清除的效率比较低下。且这种做法会让内存中的碎片非常多。这个导致了如果我们需要使用到较大的内存块时，无法分配到足够的连续内存。比如下图

![img](images/jvm知识汇总/16fa593b220bd65b)

此时可使用的内存块都是零零散散的，导致了刚刚提到的大内存对象问题

#### 3.2 复制算法

为了解决效率问题，复制算法就出现了。它将可用内存按容量划分成两等分，每次只使用其中的一块。和survivor一样也是用from和to两个指针这样的玩法。fromPlace存满了，就把存活的对象copy到另一块toPlace上，然后交换指针的内容。这样就解决了碎片的问题。

这个算法的代价就是把内存缩水了，这样堆内存的使用效率就会变得十分低下了

![img](images/jvm知识汇总/16fa599ea6f4ce56)

不过它们分配的时候也不是按照1:1这样进行分配的，就类似于Eden和Survivor也不是等价分配是一个道理。

#### 3.3 标记整理算法

复制算法在对象存活率高的时候会有一定的效率问题，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉边界以外的内存

![img](images/jvm知识汇总/16fa59da25a1629e)

#### 3.4.4 分代收集算法

这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或者“标记-整理”算法来进行回收。

说白了就是八仙过海各显神通，具体问题具体分析了而已

## 四、各种各样的垃圾回收器

![img](images/jvm知识汇总/16fa60ac474394f7)

到jdk8为止，默认的垃圾收集器是Parallel Scavenge 和 Parallel Old

从jdk9开始，G1收集器成为默认的垃圾收集器
目前来看，G1回收器停顿时间最短而且没有明显缺点，非常适合Web应用。在jdk8中测试Web应用，堆内存6G，新生代4.5G的情况下，Parallel Scavenge 回收新生代停顿长达1.5秒。G1回收器回收同样大小的新生代只停顿0.2秒。

## 五、参数说明

```
# Options that are passed to the JVM when it is launched.
JAVA_OPTS="$JAVA_OPTS -server -Xms4096m -Xmx4096m -Xmn1536m -XX:SurvivorRatio=6 -XX:MaxTenuringThreshold=6 -Xss256K -XX:LargePageSizeInBytes=128m"
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=384m"
JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection"
JAVA_OPTS="$JAVA_OPTS -XX:+UseFastAccessorMethods -XX:+UseCompressedOops -XX:+AggressiveOpts -XX:+UseBiasedLocking -XX:+DisableExplicitGC"
JAVA_OPTS="$JAVA_OPTS -verbose:gc -XX:+PrintGCDetails -XX:+PrintHeapAtGC -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution"
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${LOG_PATH}/gc/gc_dump_`date +'%Y-%m-%d_%H-%M-%S'`.bin"
JAVA_OPTS="$JAVA_OPTS -Xloggc:${LOG_PATH}/gc/gc_`date +'%Y-%m-%d_%H-%M-%S'`.log"
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"
```

| 参数                          | 含义                           | 备注                                                         |
| ----------------------------- | ------------------------------ | ------------------------------------------------------------ |
| -Xms4096m                     | 堆最小值                       |                                                              |
| -Xmx4096m                     | 堆最大值                       | 通常会将 -Xms 与 -Xmx两个参数的配置相同的值，<br />其目的是为了能够在java垃圾回收机制清理完堆区后<br />不需要重新分隔计算堆区的大小而浪费资源。 |
| -Xmn1536m                     | 年轻代大小                     | 整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。<br />增大年轻代后,将会减小年老代大小.此值对系统性能影响较大，<br />Sun官方推荐配置为整个堆的3/8 |
| -XX:SurvivorRatio=6           | 设置两个Survivor区和eden的比值 | 例如：6，表示两个Survivor:eden=2:6，即一个Survivor占年轻代的1/8 |
| -XX:MaxTenuringThreshold=6    | 设置垃圾最大年龄               |                                                              |
| -Xss256K                      | 调整每个线程栈空间的大小       | DK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。<br />在相同物理内存下,减小这个值能生成更多的线程。<br />但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右 |
| -XX:LargePageSizeInBytes=128m |                                |                                                              |
| -XX:MetaspaceSize=64m         | 初始化元空间大小               | 控制发生GC。metaSpace是不存在于JVM中的，它使用的是本地内存   <font color="red">会出现OOM吗？？？</font> |
| -XX:MaxMetaspaceSize=384m     | 限制元空间大小上限             | 防止占用过多物理内存                                         |
| -XX:+UseParNewGC | 设置年轻代为并行收集 | |
| -XX:+UseConcMarkSweepGC | 设置年老代为并发收集 | |
| -XX:CMSInitiatingOccupancyFraction=75 | 设定CMS在对内存占用率达到75%的时候开始 | |
| -XX:+UseCMSInitiatingOccupancyOnly | 只是用设定的回收阈值 | 如上面指定的75%，如果不指定，JVM仅在第一次使用设定值,后续则自动调整 |
| -XX:+CMSParallelRemarkEnabled | 降低标记停顿 | |
| -XX:+UseCMSCompactAtFullCollection | 在FULL GC的时候， 对年老代的压缩 | 设置在FULL GC的时候，对年老代的压缩CMS是不会移动内存的，因此，这个非常容易产生碎片，导致内存不够用，内存的压缩这个时候就会被启用。增加这个参数是个好习惯。可能会影响性能,但是可以消除碎片 |
| -XX:+UseFastAccessorMethods | 设置原始类型的快速优化 |  |
| -XX:+UseCompressedOops | 压缩指针 | 起到节约内存占用的参数 |
| -XX:+AggressiveOpts | 加快编译 |  |
| -XX:+UseBiasedLocking | 改善锁机制性能 |  |
| -XX:+HeapDumpOnOutOfMemoryError | 导出内存溢出的堆信息(hprof文件) |  |
| -verbose:gc |  |  |
| -XX:+PrintGCDetails | | |
| -XX:+PrintHeapAtGC | | |
| -XX:+PrintGCDateStamps | | |
| -XX:+PrintGCTimeStamps | | |
| -XX:+PrintTenuringDistribution | | |
| -XX:+PrintGCApplicationStoppedTime | | |
| -XX:HeapDumpPath=/opt/appl/tomcat/logs/gc_dump_2020-01-09_00-26-10.bin | 堆gc dump文件 | OOM排查问题使用 |
