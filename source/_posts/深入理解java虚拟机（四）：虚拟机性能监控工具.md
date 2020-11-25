---
title: 深入理解java虚拟机（四）：虚拟机性能监控工具
date: 2020-11-25 16:08:24
tags:
- java
---

##### jps：虚拟机进程状况工具

JDK的很多小工具的名字都参考了UNIX命令的命名方式，jps（JVM Process Status Tool）是其中的典型。除了名字像UNIX的ps命令之外，它的功能也和ps命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）。虽然功能比较单一，但它绝对是使用频率最高的JDK命令行工具，因为其他的JDK工具大多需要输入它查询到的LVMID来确定要监控的是哪一个虚拟机进程。对于本地虚拟机进程来说，LVMID与操作系统的进程ID（PID，Process Identifier）是一致的，使用Windows的任务管理器或者UNIX的ps命令也可以查询到虚拟机进程的LVMID，但如果同时启动了多个虚拟机进程，无法根据进程名称定位时，那就必须依赖jps命令显示主类的功能才能区分了。

jps命令格式：

`jps [ options ] [ hostid ]`

jps执行样例：

```java
D:\workspace\hello-spring-boot>jps -l
23760 com.example.demo.DemoApplication
43120
90336 org.jetbrains.jps.cmdline.Launcher
74212 sun.tools.jps.Jps
78500 org.jetbrains.idea.maven.server.RemoteMavenServer36
90276 C:\Users\jiangpeng\Desktop\jd-gui-1.4.0.jar
45496 org.jetbrains.jps.cmdline.Launcher
93068 org.jetbrains.jps.cmdline.Launcher
```

jps还可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，参数hostid为RMI注册表中注册的主机名。jps的其他常用选项：

| 序号 | 选项 | 作用                                                 |
| ---- | ---- | ---------------------------------------------------- |
| 1    | -q   | 只输出LVMID，省略主类的名称                          |
| 2    | -m   | 输出虚拟机进程启动时传递给主类main()函数的参数       |
| 3    | -l   | 输出主类的全名，如果进程执行的是jar包，输出jar包路径 |
| 4    | -v   | 输出虚拟机进程启动时的JVM参数                        |

##### jstat：虚拟机统计信息监视工具

jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据，在没有GUI图形界面、只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的常用工具。

jstat命令格式为：

`jstat [ option vmid [interval[s|ms] [count]] ]`

对于命令格式中的VMID与LVMID需要特别说明一下：如果是本地虚拟机进程，VMID与LVMID是一致的；如果是远程虚拟机进程，那VMID的格式应当是：

`[protocol:][//]lvmid[@hostname[:port]/servername]`

参数interval和count代表查询间隔和次数，如果省略这2个参数，说明只查询一次。假设需要每250毫秒查询一次进程2764垃圾收集状况，一共查询20次，那命令应当是：

`jstat -gc 2764 250 20`

选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况。常用选项：

| 序号 | 选项              | 作用                                                         |
| ---- | ----------------- | ------------------------------------------------------------ |
| 1    | -class            | 监视类装载、卸载数量、总空间以及类装载所耗费的时间           |
| 2    | -gc               | 监视Java堆状况，包括Eden区、两个Survivor区、老年代、元空间等的容量、已用空间、GC时间合计等信息 |
| 3    | -gccapacity       | 监视内容基本与-gc相同，但输出主要关注Java堆各个区域使用到的最大、最小空间 |
| 4    | -gcutil           | 监视内容基本与-gc相同，但输出主要关注已使用的空间占总空间的百分比 |
| 5    | -gccause          | 与-gcutil功能一样，但是会额外输出导致上一次GC产生的原因      |
| 6    | -gcnew            | 监视新生代GC状况                                             |
| 7    | -gcnewcapacity    | 监视内容基本与-gcnew相同，但输出主要关注使用到的最大、最小空间 |
| 8    | -gcold            | 监视老年代GC状况                                             |
| 9    | -gcoldcapacity    | 监视内容基本与-gcold相同，但输出主要关注使用到的最大、最小空间 |
| 10   | -gcpermcapacity   | 输出永久代使用到的最大、最小空间（1.8已废弃这个选项，因为永久代被移除了） |
| 11   | -compiler         | 输出即时编译器编译过的方法、耗时等信息                       |
| 12   | -printcompilation | 输出已经被即时编译的方法                                     |

我们用-gcutil来举个例子

```java
D:\workspace\hello-spring-boot>jstat -gcutil 95804
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 40.63   0.00  68.56  53.81  93.47  91.87   2430    8.866     7    2.355   11.221
```

查询结果表明，新生代Eden区（E，表示Eden）使用了68.56%的空间，两个Survivor区（S0、S1，表示Survivor0、Survivor1）一个使用了40.63%，一个是0，老年代（O，表示Old）和元空间（M。表示MetaSpace）则分别使用了53.81%和93.47%的空间。程序运行以来共发生Minor GC（YGC，表示Young GC）2430次，总共耗时8.866秒；发生Full GC（FGC，表示Full GC）7次，Full GC共耗时（FGCT，Full GC Time）为2.355秒，所有GC总耗时（GCT，表示GC Time）11.221秒。CCS表示压缩使用比例。

##### jinfo：Java配置信息工具

jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显式指定的参数列表，但如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了（如果只限于JDK6或以上版本的话，使用javaXX：+PrintFlagsFinal查看参数默认值也是一个很好的选择）。jinfo还可以使用-sysprops选项把虚拟机进程的System.getProperties()的内容打印出来。这个命令在JDK5时期已经随着Linux版的JDK发布，当时只提供了信息查询的功能，JDK6之后，jinfo在Windows和Linux平台都有提供，并且加入了在运行期修改部分参数值的能力（可以使用-flag[+|-]name或者-flag name=value在运行期修改一部分运行期可写的虚拟机参数值）。在JDK6中，jinfo对于Windows平台功能仍然有较大限制，只提供了最基本的-flag选项。

jinfo命令格式：

`jinfo [ option ] pid`

jinfo执行样例：

```java
D:\workspace\hello-spring-boot>jinfo -flag CMSInitiatingOccupancyFraction 95804
-XX:CMSInitiatingOccupancyFraction=-1
```

##### jmap：Java内存映像工具

jmap（Memory Map for Java）命令用于生成堆转储快照（一般称为heapdump或dump文件）。如果不使用jmap命令，要想获取Java堆转储快照也还有一些比较“暴力”的手段：譬如XX：+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在内存溢出异常出现之后自动生成堆转储快照文件，通过-XX：+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break]键让虚拟机生成堆转储快照文件，又或者在Linux系统下通过Kill-3命令发送进程退出信号“恐吓”一下虚拟机，也能顺利拿到堆转储快照。

jmap的作用并不仅仅是为了获取堆转储快照，它还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等。

和jinfo命令一样，jmap有部分功能在Windows平台下是受限的，除了生成堆转储快照的-dump选项和用于查看每个类的实例、空间占用统计的-histo选项在所有操作系统中都可以使用之外，其余选项都只能在Linux/Solaris中使用。

jmap命令格式：

`jmap [ option ] vmid`

常用option选项：

| 序号 | 选项           | 作用                                                         |
| ---- | -------------- | ------------------------------------------------------------ |
| 1    | -dump          | 生成Java堆转储快照。格式为-dump:[live, ]format=b,file=<filename>，其中live自参数说明是否只dump出存活的对象 |
| 2    | -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在Linux和Solaris系统下有效 |
| 3    | -heap          | 显示Java堆详细信息，如使用哪种收集器、参数配置、分代状况等。只在Linux和Solaris系统下有效 |
| 4    | -histo         | 显示堆中对象统计信息，包括类、实例数量、合计容量             |
| 5    | -permstat      | 以ClassLoader为统计口径显示永久代内存状态。只在Linux和Solaris系统下有效 |
| 6    | -F             | 当虚拟机进行对-dump选项没有响应时，可使用这个选项强制生成dump快照。只在Linux和Solaris系统下有效 |

jmap执行样例：

```java
jmap -dump:format=b,file=C:\Users\jiangpeng\Desktop\heap.hprof 95804 生dump文件
```

##### jhat：虚拟机堆转储快照分析工具

JDK提供jhat（JVM Heap Analysis Tool）命令与jmap搭配使用，来分析jmap生成的堆转储快照。jhat内置了一个微型的HTTP/Web服务器，生成堆转储快照的分析结果后，可以在浏览器中查看。不过实事求是地说，在实际工作中，除非手上真的没有别的工具可用，否则多数人是不会直接使用jhat命令来分析堆转储快照文件的，主要原因有两个方面。一是一般不会在部署应用程序的服务器上直接分析堆转储快照，即使可以这样做，也会尽量将堆转储快照文件复制到其他机器上进行分析，因为分析工作是一个耗时而且极为耗费硬件资源的过程，既然都要在其他机器上进行，就没有必要再受命令行工具的限制了。另外一个原因是jhat的分析功能相对来说比较简陋，后文将会介绍到的VisualVM，以及专业用于分析堆转储快照文件的Eclipse Memory Analyzer、IBM HeapAnalyzer等工具，都能实现 比jhat更强大专业的分析功能。

```java
jhat C:\Users\jiangpeng\Desktop\heap.hprof 分析dump文件
```

我们最好在分析前，把hprof文件放到一个单独的文件夹中，因为分析过程中会产生很多文件

```java
D:\workspace\hello-spring-boot>jhat C:\Users\jiangpeng\Desktop\11\heap.hprof
Reading from C:\Users\jiangpeng\Desktop\11\heap.hprof...
Dump file created Wed Nov 25 10:26:42 CST 2020
Snapshot read, resolving...
Resolving 50988 objects...
Chasing references, expect 10 dots..........
Eliminating duplicate references..........
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

![image-20201125170633623](https://radio93.oss-cn-beijing.aliyuncs.com/gitbub/image-20201125170633623.png?x-oss-process=style/radio93)

这个时候我们就可以用浏览器访问http://localhost:7000/了。

![image-20201125170705645](https://radio93.oss-cn-beijing.aliyuncs.com/gitbub/image-20201125170705645.png?x-oss-process=style/radio93)

分析结果默认以包为单位进行分组显示，分析内存泄漏问题主要会使用到其中的“Heap Histogram”（与jmap-histo功能一样）与OQL页签的功能，前者可以找到内存中总容量最大的对象，后者是标准的对象查询语言，使用类似SQL的语法对内存中的对象进行查询统计。

##### jstack：Java堆栈跟踪工具

jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照（一般称为threaddump或者javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因。线程出现停顿时通过jstack来查看各个线程的调用堆栈，就可以获知没有响应的线程到底在后台做些什么事情，或者等待着什么资源。

jstack命令格式：

`jstack [ option ] vmid`

option选项：

| 序号 | 选项 | 作用                                          |
| ---- | ---- | --------------------------------------------- |
| 1    | -F   | 当正常输出的请求不被响应时，强制输出线程堆栈  |
| 2    | -l   | 除堆栈外，显示关于锁的附加信息                |
| 3    | -m   | 如果调用到本地方法的时候，可以显示C/C++的堆栈 |

示例：

```java
D:\workspace\hello-spring-boot>jstack -l 95804
2020-11-25 17:11:50
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.261-b12 mixed mode):

"Keep-Alive-Timer" #17065 daemon prio=8 os_prio=1 tid=0x000001299b327800 nid=0x81cc waiting on condition [0x0000001b941ff000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at sun.net.www.http.KeepAliveCache.run(KeepAliveCache.java:172)
        at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
        - None

"XNIO-10 Accept" #1174 prio=5 os_prio=0 tid=0x000001299b32c000 nid=0x16258 runnable [0x0000001ba92fe000]
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll0(Native Method)
        at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll(WindowsSelectorImpl.java:296)
        at sun.nio.ch.WindowsSelectorImpl$SubSelector.access$400(WindowsSelectorImpl.java:278)
        at sun.nio.ch.WindowsSelectorImpl.doSelect(WindowsSelectorImpl.java:159)
        at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
        - locked <0x00000006c9f092e8> (a sun.nio.ch.Util$3)
        - locked <0x00000006c9f092d8> (a java.util.Collections$UnmodifiableSet)
        - locked <0x00000006c9efff80> (a sun.nio.ch.WindowsSelectorImpl)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:101)
        at org.xnio.nio.WorkerThread.run(WorkerThread.java:511)

   Locked ownable synchronizers:
        - None
·
·
·
"VM Thread" os_prio=2 tid=0x0000021afd95e000 nid=0xc5d8 runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000021aeac4a000 nid=0x18ae8 runnable
·
·
·
"GC task thread#7 (ParallelGC)" os_prio=0 tid=0x0000021aeac55800 nid=0x15654 runnable

"VM Periodic Task Thread" os_prio=2 tid=0x0000021a87f30000 nid=0x17df4 waiting on condition

JNI global references: 1124
```

从JDK5起，java.lang.Thread类新增了一个getAllStackTraces()方法用于获取虚拟机中所有线程的StackTraceElement对象。使用这个方法可以通过简单的几行代码完成jstack的大部分功能，在实际项目中不妨调用这个方法做个管理员页面，可以随时使用浏览器来查看线程堆栈

```java
<%@ page import="java.util.Map" %>
<html>
<head>
    <title>服务器线程信息</title>
</head>
<body>
<pre>
<%
    for (Map.Entry<Thread, StackTraceElement[]> stackTrace : Thread.getAllStackTraces().entrySet()) {
        Thread thread = (Thread) stackTrace.getKey();
        StackTraceElement[] stack = (StackTraceElement[]) stackTrace.getValue();
        if (thread.equals(Thread.currentThread())) {
            continue;
        }
        System.out.print("\n线程：" + thread.getName() + "\n");
        for (StackTraceElement element : stack) {
            System.out.print("\t" + element + "\n");
        }
    }
%>
</pre>
</body>
</html>
```

```java
线程：http-nio-9000-exec-6
	sun.misc.Unsafe.park(Native Method)
	java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:107)
	org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:33)
	java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	java.lang.Thread.run(Thread.java:748)

线程：RMI TCP Connection(2)-192.168.32.16
	java.net.SocketInputStream.socketRead0(Native Method)
	java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	java.net.SocketInputStream.read(SocketInputStream.java:171)
	java.net.SocketInputStream.read(SocketInputStream.java:141)
	java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	java.io.BufferedInputStream.read(BufferedInputStream.java:265)
	java.io.FilterInputStream.read(FilterInputStream.java:83)
	sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:555)
	sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:834)
	sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.lambda$run$0(TCPTransport.java:688)
	sun.rmi.transport.tcp.TCPTransport$ConnectionHandler$$Lambda$93/795161234.run(Unknown Source)
	java.security.AccessController.doPrivileged(Native Method)
	sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:687)
	java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	java.lang.Thread.run(Thread.java:748)
```

##### 其他工具汇总

性能监控和故障处理：用于监控分析Java虚拟机运行信息

![image-20201125172630917](https://radio93.oss-cn-beijing.aliyuncs.com/gitbub/image-20201125172630917.png?x-oss-process=style/radio93)

