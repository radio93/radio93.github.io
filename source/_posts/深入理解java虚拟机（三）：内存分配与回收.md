---
title: 深入理解java虚拟机（三）：内存分配与回收
date: 2020-11-24 17:52:27
tags:
- java
---

在这一节开始前，我们先介绍总结一下垃圾收集器的一些参数

| 序号 | 参数                               | 描述                                                         |
| ---- | ---------------------------------- | ------------------------------------------------------------ |
| 1    | -XX:+UseSerialGC                   | Jvm运行在Client模式下的默认值，打开此开关后，使用Serial + Serial Old的收集器组合进行内存回收 |
| 2    | -XX:+UseParNewGC                   | 打开此开关后，使用ParNew + Serial Old的收集器进行垃圾回收    |
| 3    | -XX:+UseConcMarkSweepGC            | 使用ParNew + CMS +  Serial Old的收集器组合进行内存回收，Serial Old作为CMS出现“Concurrent Mode Failure”失败后的后备收集器使用。 |
| 4    | -XX:+UseParallelGC                 | Jvm运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge + Serial Old的收集器组合进行回收 |
| 5    | -XX:+UseParallelOldGC              | 使用Parallel Scavenge + Parallel Old的收集器组合进行回收 ==**(JDK1.8默认)**== |
| 6    | -XX:SurvivorRatio                  | 新生代中Eden区域与Survivor区域的容量比值，默认为8，代表Eden:Subrvivor = 8:1 |
| 7    | -XX:PretenureSizeThreshold         | 直接晋升到老年代对象的大小，设置这个参数后，大于这个参数的对象将直接在老年代分配 |
| 8    | -XX:MaxTenuringThreshold           | 晋升到老年代的对象年龄，每次Minor GC之后，年龄就加1，当超过这个参数的值时进入老年代 |
| 9    | -XX:UseAdaptiveSizePolicy          | 动态调整java堆中各个区域的大小以及进入老年代的年龄           |
| 10   | -XX:+HandlePromotionFailure        | 是否允许新生代收集担保，进行一次minor gc后,另一块Survivor空间不足时，将直接会在老年代中保留 |
| 11   | -XX:ParallelGCThreads              | 设置并行GC进行内存回收的线程数                               |
| 12   | -XX:GCTimeRatio                    | GC时间占总时间的比列，默认值为99，即允许1%的GC时间，仅在使用Parallel Scavenge收集器时有效 |
| 13   | -XX:MaxGCPauseMillis               | 设置GC的最大停顿时间，在Parallel Scavenge收集器下有效        |
| 14   | -XX:CMSInitiatingOccupancyFraction | 设置CMS收集器在老年代空间被使用多少后出发垃圾收集，默认值为68%，仅在CMS收集器时有效，-XX:CMSInitiatingOccupancyFraction=70 |
| 15   | -XX:+UseCMSCompactAtFullCollection | 由于CMS收集器会产生碎片，此参数设置在垃圾收集器后是否需要一次内存碎片整理过程，仅在CMS收集器时有效 |
| 16   | -XX:+CMSFullGCBeforeCompaction     | 设置CMS收集器在进行若干次垃圾收集后再进行一次内存碎片整理过程，通常与UseCMSCompactAtFullCollection参数一起使用 |
| 17   | -XX:+UseFastAccessorMethods        | 原始类型优化                                                 |
| 18   | -XX:+DisableExplicitGC             | 是否关闭手动System.gc                                        |
| 19   | -XX:+CMSParallelRemarkEnabled      | 降低标记停顿                                                 |
| 20   | -XX:LargePageSizeInBytes           | 内存页的大小不可设置过大，会影响Perm的大小，-XX:LargePageSizeInBytes=128m |

下面的例子我们都采用Parallel Scavenge + Parallel Old的收集器组合进行内存回收（即-XX:+UseParallelOldGC）；真实情况，可能会各种各样，可以再具体分析。

##### 对象优先在Eden分配

大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

HotSpot虚拟机提供了`-XX：+PrintGCDetails`这个收集器日志参数，告诉虚拟机在发生垃圾收集行为时打印内存回收日志，并且在进程退出的时候输出当前的内存各区域分配情况。在实际的问题排查中，收集器日志常会打印到文件后通过工具进行分析。

```java
//VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
public static void main(String[] args) {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[2 * _1MB];
    allocation2 = new byte[2 * _1MB];
    allocation3 = new byte[2 * _1MB];
    allocation4 = new byte[4 * _1MB]; // 出现一次Minor GC
}
```

```java
[GC (Allocation Failure) [PSYoungGen: 6738K->993K(9216K)] 6738K->1831K(19456K), 0.0029895 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 7459K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 78% used [0x00000000ff600000,0x00000000ffc507b8,0x00000000ffe00000)
  from space 1024K, 97% used [0x00000000ffe00000,0x00000000ffef8738,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 4933K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 48% used [0x00000000fec00000,0x00000000ff0d1640,0x00000000ff600000)
 Metaspace       used 3376K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 365K, capacity 388K, committed 512K, reserved 1048576K
```

我们来分析一下这个垃圾回收的日志信息

这里我们先说明下几个收集器日志

> DefNew：是使用-XX:+UseSerialGC（新生bai代，老年代都使用串行回收收集器）。 
>
> 并行收集器：
>
> ​	ParNew：是使用-XX:+UseParNewGC（新生代使用并行收集器，老年代使用串行回收收集器）或者-XX:+UseConcMarkSweepGC(新生代使用并行收集器，老年代使用CMS)。
>
> ​	PSYoungGen：是使用-XX:+UseParallelOldGC（新生代，老年代都使用并行回收收集器）或者-XX:+UseParallelGC（新生代使用并行回收收集器，老年代使用串行收集器）
>
> ​	garbage-first heap：是使用-XX:+UseG1GC（G1收集器）

```java
     	回收前  回收后（新生代的总内存） 堆回收前  堆回收后（堆的总内存）
[PSYoungGen: 6738K->993K(9216K)] 6738K->1831K(19456K), 0.0029895 secs]
```

尝试分配三个2MB大小和一个4MB大小的对象，在运行时通过-Xms20M、-Xmx20M、-Xmn10M这三个参数限制了Java堆大小为20MB，不可扩展，其中10MB分配给新生代，剩下的10MB分配给老年代。-XX：Survivor-Ratio=8决定了新生代中Eden区与一个Survivor区的空间比例是8∶1，从输出的结果也清晰地看到“eden space 8192K、from space 1024K、to space 1024K”的信息，新生代总可用空间为9216KB（Eden区+1个Survivor区的总容量）。

分配allocation4对象的语句时会发生一次Minor GC，这次回收的结果是新生代6738KB变为993KB，而总内存占用量则几乎没有减少（因为allocation1、2、3三个对象都是存活的，虚拟机几乎没有找到可回收的对象）。产生这次垃圾收集的原因是为allocation4分配内存时，发现Eden已经被占用了6MB，剩余空间已不足以分配allocation4所需的4MB内存，因此发生Minor GC。垃圾收集期间虚拟机又发现已有的三个2MB大小的对象全部无法放入Survivor空间（Survivor空间只有1MB大小），所以只好通过分配担保机制提前转移到老年代去。

这次收集结束后，4MB的allocation4对象顺利分配在Eden中。因此程序执行完的结果是Eden占用4MB（被allocation4占用），Survivor空闲，老年代被占用6MB（被allocation1、2、3占用）。

##### 大对象直接进入老年代

大对象就是指需要大量连续内存空间的Java对象，最典型的大对象便是那种很长的字符串，或者元素数量很庞大的数组，例子中的byte[]数组就是典型的大对象。大对象对虚拟机的内存分配来说就是一个不折不扣的坏消息，比遇到一个大对象更加坏的消息就是遇到一群“朝生夕灭”的“短命大对象”，我们写程序的时候应注意避免。在Java虚拟机中要避免大对象的原因是，在分配空间时，它容易导致内存明明还有不少空间时就提前触发垃圾收集，以获取足够的连续空间才能安置好它们，而当复制对象时，大对象就意味着高额的内存复制开销。HotSpot虚拟机提供了-XX：PretenureSizeThreshold参数，指定大于该设置值的对象直接在老年代分配，这样做的目的就是避免在Eden区及两个Survivor区之间来回复制，产生大量的内存复制操作。

>-XX：PretenureSizeThreshold参数只对Serial和ParNew两款新生代收集器有效，HotSpot的其他新生代收集器，如Parallel Scavenge并不支持这个参数。如果必须使用此参数进行调优，可考虑ParNew加CMS的收集器组合。

```java
//-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:PretenureSizeThreshold=3145728 
//-XX:+UseParNewGC
public static void main(String[] args) {
    byte[] allocation1;
    allocation1 = new byte[4 * _1MB]; // 出现一次Minor GC
}
```

```java
Heap
 par new generation   total 9216K, used 7082K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  86% used [0x00000000fec00000, 0x00000000ff2ea820, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 concurrent mark-sweep generation total 10240K, used 4096K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3455K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 373K, capacity 388K, committed 512K, reserved 1048576K
```

##### 长期存活的对象将进入老年代

HotSpot虚拟机中多数收集器都采用了分代收集来管理堆内存，那内存回收时就必须能决策哪些存活对象应当放在新生代，哪些存活对象放在老年代中。为做到这点，虚拟机给每个对象定义了一个对象年龄（Age）计数器，存储在对象头中（详见第2章）。对象通常在Eden区里诞生，如果经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，该对象会被移动到Survivor空间中，并且将其对象年龄设为1岁。对象在Survivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15），就会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数-XX:MaxTenuringThreshold设置。

##### 动态对象年龄判定

为了能更好地适应不同程序的内存状况，HotSpot虚拟机并不是永远要求对象的年龄必须达到XX:MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到-XX:MaxTenuringThreshold中要求的年龄。

##### 空间分配担保

在发生Minor GC之前，虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那这一次Minor GC可以确保是安全的。如果不成立，则虚拟机会先查看XX:HandlePromotionFailure参数的设置值是否允许担保失败（Handle Promotion Failure）；如果允许，那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者-XX: HandlePromotionFailure设置不允许冒险，那这时就要改为进行一次Full GC。

解释一下“冒险”是冒了什么风险：前面提到过，新生代使用复制收集算法，但为了内存利用率，只使用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况——最极端的情况就是内存回收后新生代中所有对象都存活，需要老年代进行分配担保，把Survivor无法容纳的对象直接送入老年代，这与生活中贷款担保类似。老年代要进行这样的担保，前提是老年代本身还有容纳这些对象的剩余空间，但一共有多少对象会在这次回收中活下来在实际完成内存回收之前是无法明确知道的，所以只能取之前每一次回收晋升到老年代对象容量的平均大小作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间。

##### 小结

垃圾收集器在许多场景中都是影响系统停顿时间和吞吐能力的重要因素之一，虚拟机之所以提供多种不同的收集器以及大量的调节参数，就是因为只有根据实际应用需求、实现方式选择最优的收集方式才能获取最好的性能。没有固定收集器、参数组合，没有最优的调优方法，虚拟机也就没有什么必然的内存回收行为。