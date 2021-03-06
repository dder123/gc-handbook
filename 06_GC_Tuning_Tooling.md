# 6. GC 调优(工具篇)


Before you can optimize your JVM for more efficient garbage collection, you need to get information about its current behavior to understand the impact that GC has on your application and its perception by end users. There are multiple ways to observe the work of GC and in this chapter we will cover several different possibilities.

在调优JVM的GC性能之前, 需要明确知道, 当前的GC行为对系统和用户有多大的影响. 有很多监控GC行为的方式, 本章介绍常用的工具。


While observing GC behavior there is raw data provided by the JVM runtime. In addition, there are derived metrics that can be calculated based on that raw data. The raw data, for instance, contains:

在程序运行的过程中,JVM提供了GC相关行为的原始数据。另外, 可以利用原始数据生成各种报告。原始数据(raw data)包含以下部分:


- current occupancy of memory pools,
- capacity of memory pools,
- durations of individual GC pauses,
- duration of the different phases of those pauses.

<br/>

- 当前各内存池的使用情况,
- 各个内存池的容量,
- 每次GC暂停的持续时间,
- GC暂停各个阶段的持续时间。


The derived metrics include, for example, the allocation and promotion rates of the application. In this chapter will talk mainly about ways of acquiring the raw data. The most important derived metrics are described in the following chapter discussing the most common GC-related performance problems.

可以计算出各种指标, 例如: 程序的内存分配率和提升率。本章主要介绍获取原始数据的方法. 后续章节将讨论最重要的派生指标(derived metrics)，以及GC性能相关的问题。


## JMX API


The most basic way to get GC-related information from the running JVM is via the standard [JMX API](https://docs.oracle.com/javase/tutorial/jmx/index.html). This is a standardized way for the JVM to expose internal information regarding the runtime state of the JVM. You can access this API either programmatically from your own application running inside the very same JVM, or using JMX clients.

从 JVM 运行时获取GC相关(GC-related)信息, 最基本的方式是通过标准 [JMX API 接口](https://docs.oracle.com/javase/tutorial/jmx/index.html). JMX是展示JVM内部运行时相关状态信息的标准API. 我们可以通过编程的方式,通过 JMX API 访问运行本程序的JVM，还可以通过JMX客户端来(远程)访问。



Two of the most popular JMX clients are [JConsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html) and [JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) (with a corresponding plugin installed). Both of these tools are part of the standard JDK distribution, so getting started is easy. If you are running on JDK 7u40 or later, a third tool bundled into the JDK called Java Mission Control is also available.

最常见的两种JMX客户端是 [JConsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html) 和 [JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) (可以安装各种插件,十分强大)。这两个工具都是标准JDK的一部分, 很容易使用. 如果使用的是 JDK 7u40 及更新的版本,还可以使用第三种工具: [Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)( 大致翻译为 Java控制中心, `jmc.exe`)。

> JVisualVM安装MBeans插件,通过 工具(T) -- 插件(G) -- 可用插件 -- 勾选VisualVM-MBeans -- 安装 -- 下一步 -- 等待安装完成... 其他的插件安装也类似。


All the JMX clients run as a separate application connecting to the target JVM. The target JVM can be either local to the client if running both in the same machine, or remote. For the remote connections from client, JVM has to explicitly allow remote JMX connections. This can be achieved by setting a specific system property to the port where you wish to enable the JMX RMI connection to arrive:

所有的JMX客户端都是独立的程序,可以连接到目标JVM。目标JVM可以是本地JVM, 也可以是远程JVM. 如果要连接远程JVM, 则目标JVM必须通过特定的环境变量参数来启动,以开启远程JMX连接。 开启远程JMX RMI连接, 并指定端口号的示例如下:


	java -Dcom.sun.management.jmxremote.port=5432 com.yourcompany.YourApp



In the example above, the JVM opens port 5432 for JMX connections.

在此示例中, JVM 打开端口5432以支持JMX连接。


After connecting your JMX client to the JVM of interest and navigating to the MBeans list, select MBeans under the node “java.lang/GarbageCollector”. See below for two screenshots exposing information about GC behavior from JVisualVM and Java Mission Control, respectively:

通过 JVisualVM  连接到某个JVM之后, 导航到 MBeans list, 展开 “java.lang/GarbageCollector” 下的 MBeans. 下图显示了 JVisualVM 和Java Mission Control 中的GC行为信息:


![](06_01_JMX-view.png)




![](06_02_JMX-view-Mbean.png)




As the screenshots above indicate, there are two garbage collectors present. One of these collectors is responsible for cleaning the young generation and one for the old generation. The names of those elements correspond to the names of the garbage collectors used. In the screenshots above we can see that the particular JVM is running with ParallelNew for the young generation and with Concurrent Mark and Sweep for the old generation.

上面的截图信息中, 共有两种垃圾收集器。其中一种负责清理年轻代，另一种负责清理老年代. 图中两个元素的名字就是的垃圾收集器名称. 从图中可以看到, 此JVM中, 年轻代垃圾收集器是 **PS Scavenge**; 老年代使用的是 **PS MarkSweep** 算法。


For each collector the JMX API exposes the following information:

对每一款垃圾收集器, 通过 JMX API 展示的信息以下:


- CollectionCount – the total number of times this collector has run in this JVM,
- CollectionTime – the accumulated duration of the collector running time. The time is the sum of the wall-clock time of all GC events,
- LastGcInfo – detailed information about the last garbage collection event. This information contains the duration of that event, and the start and end time of the event along with the usage of different memory pools before and after the last collection,
- MemoryPoolNames – names of the memory pools that this collector manages,
- Name – the name of the garbage collector
- ObjectName – the name of this MBean, as dictated by JMX specifications,
- Valid – shows whether this collector is valid in this JVM. I personally have never seen anything but “true” in here

<br/>

- **CollectionCount** : 此垃圾收集器执行GC的总次数,
- **CollectionTime**: 收集器运行时间的累计。此时间是所有GC事件的时间总和,
- **LastGcInfo**: 最近一次GC事件的详细信息。包括GC事件的 持续时间(duration),  开始时间(startTime) 和 结束时间(endTime), 以及各个内存池在最近一次GC之前和之后的使用情况,
- **MemoryPoolNames**:  此款垃圾收集器所管理内存池的名称,
- **Name**: 垃圾收集器的名称
- **ObjectName**: 由JMX规范定义的 MBean的名字,,
- **Valid**: 此收集器是否有效。本人只见过为 "true"的情况


In my experience this information is not enough to make any conclusions about the efficiency of the garbage collector. The only case where it can be of any use is when you are willing to build custom software to get JMX notifications about garbage collection events. This approach can rarely be used as we will see in the next sections, which give better ways of getting beneficial insight into garbage collection activities.

根据经验, 这些信息对GC的性能来说,不能得出什么结论. 唯一可行的方式, 是自己编写程序, 通过获取GC相关的 JMX 通知来进行统计。 从下一节可以看到, 一般也不怎么查看 MBean , 但对于理解GC活动倒是挺有用的。



## JVisualVM


[JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) adds extra information to the basic JMX client functionality via a separate plugin called “[VisualGC](http://www.oracle.com/technetwork/java/visualgc-136680.html)”. It provides a real-time view into GC events and the occupancy of different memory regions inside JVM.

[JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html) 通过 “[VisualGC](http://www.oracle.com/technetwork/java/visualgc-136680.html)” 插件提供JMX客户端的基本功能, 此外还提供了实时界面, 展示 GC事件以及JVM各个内存空间的使用情况。


The most common use-case for the Visual GC plugin is the monitoring of the locally running application, when an application developer or a performance specialist wants an easy way to get visual information about the general behavior of the GC during a test run of the application.

Visual GC 插件最常用的功能, 是监控本机运行的Java程序, 比如开发者和性能调优专家,想要快速获取测试程序的GC相关信息时, 就会很有用。


![](06_03_jvmsualvm-garbage-collection-monitoring.png)



On the left side of the charts you can see the real-time view of the current usages of the different memory pools: Metaspace or Permanent Generation, Old Generation, Eden Generation and two Survivor Spaces.

左边的图表部分,可以看到各个内存池的使用情况: Metaspace/永久代, 老年代, Eden区以及两个存活区。


On the right side, top two charts are not GC related, exposing JIT compilation times and class loading timings. The following six charts display the history of the memory pools usages, the number of GC collections of each pool and cumulative time of GC for that pool. In addition for each pool its current size, peak usage and maximum size are displayed.

在右边, 顶部的两个图表与 GC无关, 分别显示的是 JIT编译时间 和 类加载时间。下面的6个图显示的是内存池使用情况的历史记录, 每个内存池对应的GC次数,GC总时间, 还有每个内存池的最大值，峰值, 及当前使用情况。


Below is the distribution of objects ages that currently reside in the Young generation. The full discussion of objects tenuring monitoring is outside of the scope of this chapter.

再下面是 HistoGram, 显示的是年轻代中的对象年龄分布。关于对象的年龄监控(objects tenuring monitoring), 超出了本章的范围, 不再进行描述。


When compared with pure JMX tools, the VisualGC add-on to JVisualVM does offer slightly better insight to the JVM, so when you have only these two tools in your toolkit, pick the VisualGC plug-in. If you can use any other solutions referred to in this chapter, read on. Alternative options can give you more information and better insight. There is however a particular use-case discussed in the “Profilers” section where JVisualVM is suitable for – namely allocation profiling, so by no means we are demoting the tool in general, just for the particular use case.

与纯粹的JMX工具相比, VisualGC 插件对 JVM 的内部信息提供了更直观的界面, 如果暂时没有其他工具, 请选择VisualGC插件. 本章接下来会继续介绍其他工具, 这些工具可以提供更多的信息, 以及更好的视角. 当然, 在“Profilers(分析器)”一节中，也会介绍 JVisualVM 的适用场景 —— 如: 分配分析(allocation profiling), 所以我们绝不是在贬低哪一款工具, 关键还得看实际情况。




## jstat



The next tool to look at is also part of the standard JDK distribution. The tool is called “[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)” – a Java Virtual Machine statistics monitoring tool. This is a command line tool that can be used to get metrics from the running JVM. The JVM connected can again either be local or remote. A full list of metrics that jstat is capable of exposing can be obtained by running “jstat -option” from the command line. The most commonly used options are:

[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html) 也是标准JDK提供的一款JVM统计监控工具(Java Virtual Machine statistics monitoring tool). jstat 可以从JVM中获取各种指标。既可以连接到本地JVM,也可以连到远程JVM. 可以执行 “`jstat -options`” 来查看支持的指标和对应选项。常用的包括:


	+-----------------+---------------------------------------------------------------+
	|     Option      |                          Displays...                          |
	+-----------------+---------------------------------------------------------------+
	|class            | Statistics on the behavior of the class loader                |
	|compiler         | Statistics  on  the behavior of the HotSpot Just-In-Time com- |
	|                 | piler                                                         |
	|gc               | Statistics on the behavior of the garbage collected heap      |
	|gccapacity       | Statistics of the capacities of  the  generations  and  their |
	|                 | corresponding spaces.                                         |
	|gccause          | Summary  of  garbage collection statistics (same as -gcutil), |
	|                 | with the cause  of  the  last  and  current  (if  applicable) |
	|                 | garbage collection events.                                    |
	|gcnew            | Statistics of the behavior of the new generation.             |
	|gcnewcapacity    | Statistics of the sizes of the new generations and its corre- |
	|                 | sponding spaces.                                              |
	|gcold            | Statistics of the behavior of the old and  permanent  genera- |
	|                 | tions.                                                        |
	|gcoldcapacity    | Statistics of the sizes of the old generation.                |
	|gcpermcapacity   | Statistics of the sizes of the permanent generation.          |
	|gcutil           | Summary of garbage collection statistics.                     |
	|printcompilation | Summary of garbage collection statistics.                     |
	+-----------------+---------------------------------------------------------------+




This tool is extremely useful for getting a quick overview of JVM health to see whether the garbage collector behaves as expected. You can run it via “jstat -gc -t PID 1s”, for example, where PID is the process ID of the JVM you want to monitor. You can acquire PID via running “jps” to get the list of running Java processes. As a result, each second jstat will print a new line to the standard output similar to the following example:

这款工具对于快速查看GC行为是否健康运行是很有用的。启动方式为: “`jstat -gc -t PID 1s`” , 其中,PID 就是要监视的JVM进程ID。正在运行的Java进程可用通过 `jps` 命令得到。


	jps

	jstat -gc -t 2428 1s


以上命令的结果, 是 jstat 每秒向标准输出输出一行新内容, 示例如下:


	Timestamp  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
	200.0  	 8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169344.0  21248.0 20534.3 3072.0 2807.7     34    0.720  658   133.684  134.404
	201.0 	 8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169343.2  21248.0 20534.3 3072.0 2807.7     34    0.720  662   134.712  135.432
	202.0 	 8448.0 8448.0 8102.5  0.0   67712.0  67598.5   169344.0   169343.6  21248.0 20534.3 3072.0 2807.7     34    0.720  667   135.840  136.559
	203.0 	 8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
	204.0 	 8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
	205.0 	 8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
	206.0 	 8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
	207.0 	 8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009
	208.0 	 8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009




Let us interpret the output above using the explanation given to output attributes in the [jstat manpage](http://www.manpagez.com/man/1/jstat/). Using the knowledge acquired, we can see that:

稍微解释一下上面的内容。通过以上信息, 参考 [jstat manpage](http://www.manpagez.com/man/1/jstat/) , 我们可以知道:


- jstat connected to the JVM 200 seconds from the time this JVM was started. This information is present in the first column labeled “Timestamp”. As seen from the very same column, the jstat harvests information from the JVM once every second as specified in the “1s” argument given in the command.
- From the first line we can see that up to this point the young generation has been cleaned 34 times and the whole heap has been cleaned 658 times, as indicated by the “YGC” and “FGC” columns, respectively.
- The young generation garbage collector has been running for a total of 0.720 seconds, as indicated in the “YGCT” column.
- The total duration of the full GC has been 133.684 seconds, as indicated in the “FGCT” column. This should immediately catch our eye – we can see that out of the total 200 seconds the JVM has been running, **66% of the time has been spent in Full GC cycles**.

<br/>

- jstat 连接到 JVM 的时间, 是此JVM启动后的 200秒。此信息由第一行的 “Timestamp” 列得知。继续查看下一行, jstat 每秒钟从JVM 接收一次信息, 也就是命令行参数中 "`1s`" 的意思。
- 从第一行我们还可以看到, 年轻代共执行了34次清理(由 “**YGC**” 列得知), 整个堆内存已经执行了 658次清理(由  “**FGC**” 列得知)。
- 年轻代的垃圾收集的总耗时时间为 0.720 秒, 显示在“**YGCT**” 这一列。
- Full GC 的总耗时时间为 133.684 秒, 由“**FGCT**”列表示. 这立马就引起了我们的注意, 可以看到,JVM 总的只运行了 200 秒的时间, **但其中 66% 的部分被 Full GC 占用了**。


The problem becomes even clearer when we look at the next line, harvesting information a second later.

我们再看下一行, 问题就更明显了。




- Now we can see that there have been four more Full GCs running during the one second between the last time jstat printed out the data as indicated in the “FGC” column.
- These four GC pauses have taken almost the entire second – as seen in the difference in the “FGCT” column. Compared to the first row, the Full GC has been running for **928 milliseconds**, or 92.8% of the total time.
- At the same time, as indicated by the “OC” and “OU” columns, we can see that from almost all of the old generation capacity of 169,344.0 KB (“OC“), after all the cleaning work that the four collection cycles tried to accomplish, 169,344.2 KB (“OU“) is still in use. Cleaning 800 bytes in 928 ms should not be considered a normal behavior.

<br/>

- 可以看到, 在接下来的一秒之内共执行了 4 次Full GC。参见 "**FGC**" 列.
- 这4次GC暂停几乎占用了整整 1秒的时间(根据 **FGCT**列的差得知)。与第一行相比,  Full GC 运行了`928 毫秒`, 也就是 `92.8%` 的时间。
- 与此同时, 根据 “**OC** 和 “**OU**” 列, 可以得知, **整个老年代的空间**为 169,344.0 KB (“OC“), 在 4 次GC之后依然占用了 169,344.2 KB (“OU“)。在 928ms 的时间内只释放了 800个字节, 怎么看都觉得不正常。


Only these two rows from the jstat output give us insight that something is terribly wrong with the application. Applying the same analytics to the next rows, we can confirm that the problem persists and is getting even worse.

只看这两行输出内容, 就知道应用程序出了些严重的问题。继续分析下一行, 可以确定问题依然存在,而且变得更糟。



The JVM is almost stalled, with GC eating away more than 90% of the available computing power. And as a result of all this cleaning, almost all the old generation still remains in use, further confirming our doubts. As a matter of fact, the example died in under a minute later with a “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” error, removing the last remaining doubts whether or not things are truly sour.

JVM几乎完全卡住了(stalled), 因为GC占用了超过90%的计算资源。清理之后, 所有的老代空间还在占用着, 这进一步证实了我们的猜测。事实上, 这个程序在运行一分钟以后就挂了, 抛出了一个 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)”  错误。


As seen from the example, jstat output can quickly reveal symptoms about JVM health in terms of misbehaving garbage collectors. As general guidelines, just looking at the jstat output will quickly reveal the following symptoms:

从示例可以看到, 通过 jstat 的输出内容可以快速发现对JVM健康极为不利的GC行为。一般来说, 只看 jstat 的输出可以很快发现以下问题:


- Changes in the last column, “GCT”, when compared to the total runtime of the JVM in the “Timestamp” column, give information about the overhead of garbage collection. If you see that every second, the value in that column increases significantly in comparison to the total runtime, a high overhead is exposed. How much GC overhead is tolerable is application-specific and should be derived from the performance requirements you have at hand, but as a ground rule, anything more than 10% looks truly suspicious.
- Rapid changes in the “YGC” and “FGC” columns tracking the young and Full GC counts also tend to expose problems. Much too frequent GC pauses when piling up again affect the throughput via adding many stop-the-world pauses for application threads.
- When you see that the old generation usage in the “OU” column is almost equal to the maximum capacity of the old generation in the “OC” column without a decrease after an increase in the “FGC” column count has signaled that old generation collection has occurred, you have exposed yet another symptom of poorly performing GC.

<br/>

- 最后一列 “**GCT**”, 与JVM的总运行时间 “**Timestamp**” 的比值, 就是GC 的开销。如果每一秒中, "**GCT**" 的值都会显著增加, 与总运行时间相比, 就暴露出GC高开销的事实. 具体多少比例的GC开销是可容忍的, 不同的系统有不同的容忍度, 具体是由性能需求而决定, 但作为一般原则, 超过 10% 的GC开销看起来都是有问题的。
- “**YGC**” 和 “**FGC**” 列的快速变化往往也是有问题的. 太频繁的GC暂停会积累并导致更多的线程停顿(stop-the-world pauses),影响到吞吐量。
- 如果看到 “**OU**” 列中,老年代的使用量大约等于老年代的最大容量(**OC**), 而且不会降低, 那就表示虽然执行了老年代GC, 但GC的性能非常烂。



## GC日志(GC logs)


The next source for GC-related information is accessible via garbage collector logs. As they are built in the JVM, GC logs give you (arguably) the most useful and comprehensive overview about garbage collector activities. GC logs are de facto standard and should be referenced as the ultimate source of truth for garbage collector evaluation and optimization.

通过GC日志也可以获取垃圾收集相关的信息。因为JVM内置了GC日志模块, 所以对垃圾收集活动最全面最有用的内容都包含在GC日志里. GC日志是事实上的标准, 可以作为对垃圾收集评估和优化的最真实数据。


A garbage collector log is in plain text and can be either printed out by the JVM to standard output or redirected to a file. There are many JVM options related to GC logging. For example, you can log the total time for which the application was stopped before and during each GC event (-XX:+PrintGCApplicationStoppedTime) or expose the information about different reference types being collected (-XX:+PrintReferenceGC).

GC日志是纯文本格式的,一般输出到文件之中, 当然也可以打印到控制台。有很多个JVM参数可以控制GC的日志。例如,可以打印每次GC事件的持续时间, 以及程序暂停了多久(`-XX:+PrintGCApplicationStoppedTime`), 还有垃圾回收清理多少引用类型(`-XX:+PrintReferenceGC`)。


The minimum that each JVM should be logging can be achieved by specifying the following in your startup scripts:

要记录GC日志, 需要在启动脚本设置下面这样的JVM参数:


	-XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:<filename>



This will instruct JVM to print every GC event to the log file and add the timestamp of each event to the log. The exact information exposed to the logs varies depending on the GC algorithm used. When using ParallelGC the output should look similar to the following:

以上参数指示JVM: 将所有的GC事件打印到日志文件, 并打印每次事件的日期和时间戳。具体的输出内容, 根据GC算法不同而略有不同. ParallelGC 的输出内容示例如下:


	199.879: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1473386 secs] [Times: user=0.43 sys=0.01, real=0.15 secs]
	200.027: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1567794 secs] [Times: user=0.41 sys=0.00, real=0.16 secs]
	200.184: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1621946 secs] [Times: user=0.43 sys=0.00, real=0.16 secs]
	200.346: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1547695 secs] [Times: user=0.41 sys=0.00, real=0.15 secs]
	200.502: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1563071 secs] [Times: user=0.42 sys=0.01, real=0.16 secs]
	200.659: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1538778 secs] [Times: user=0.42 sys=0.00, real=0.16 secs]




These different formats are discussed in detail in the chapter “[GC Algorithms: Implementations](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations)”, so if you are not familiar with the output, please read this chapter first. If you can already interpret the output above, then you are able to deduct that:

这些不同的格式在 “[04 GC算法:实现篇](http://blog.csdn.net/renfufei/article/details/54885190)” 中详细讨论过了,如果对此不了解, 可以先阅读第4章. 解析以上输出内容,可以得知:


- The log is extracted around 200 seconds after the JVM was started.
- During the 780 milliseconds present in the logs, the JVM paused five times for GC (we are excluding the 6th pause as it started, not ended on the timestamp present). All these pauses were full GC pauses.
- The total duration of these pauses was 777 milliseconds, or 99.6% of the total runtime.
- At the same time, as seen from the old generation capacity and consumption, almost all of the old generation capacity (169,472 kB) remains used (169,318 K) after the GC has repeatedly tried to free up some memory.

<br/>

- 这部分日志截取自JVM启动后200秒左右。
- 日志片段中显示, 在780毫秒之内, JVM因为GC停顿了五次(去掉第六次暂停,这样更精确一些). 所有停顿都是 Full GC暂停。
- 这些暂停事件的总持续时间是 777毫秒, 占总运行时间的 **99.6%**。
- 与此同时, 可以看到老年代的容量与使用情况, 在GC完成之后,几乎所有的老年代空间(`169,472 KB`)仍被占用(`169,318 K`)。


From the output, we can confirm that the application is not performing well in terms of GC. The JVM is almost stalled, with GC eating away more than 99% of the available computing power. And as a result of all this cleaning, almost all the old generation still remains in use, further confirming our suspicions. The example application, being the same as used in the previous jstat section, died in just a few minutes later with the “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” error, confirming that the problem was severe.


通过日志信息可以确认, 该应用的GC状况非常恶劣。JVM几乎处于停滞状态, 因为GC占用了超过99%的CPU时间. 而垃圾回收的结果, 是整个老年代空间仍然被占用, 这进一步证实了我们的猜测。 和jstat 小节中是同一个示例程序, 几分钟之后程序就挂了, 抛出的也是 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误, 确认问题是很严重的.


As seen from the example, GC logs are valuable input to reveal symptoms about JVM health in terms of misbehaving garbage collectors. As general guidelines, the following symptoms can quickly be revealed by just looking at GC logs:

从这个例子中可以看出, GC日志对监控JVM的GC行为状态是否健康非常有价值。一般情况下, 查看 GC 日志就可以很快确定以下症状:


- Too much GC overhead. When the total time for GC pauses is too long, the throughput of the application suffers. The limit is application-specific, but as a general rule anything over 10% already looks suspicious.
- Excessively long individual pauses. Whenever an individual pause starts taking too long, the latency of the application starts to suffer. When the latency requirements require the transactions in the application to complete under 1,000 ms, for example, you cannot tolerate any GC pauses taking more than 1,000 ms.
- Old generation consumption at the limits. Whenever the old generation remains close to being fully utilized even after several full GC cycles, you face a situation in which GC becomes the bottleneck, either due to under-provisioning resources or due to a memory leak. This symptom always triggers GC overhead to go through the roof as well.

<br/>

- GC开销太大。如果GC暂停的总时间太长, 系统的吞吐量就会受到损害。具体允许多大比例由系统特性而确定, 但超过 10% 的开销一般认为是不正常的。
- 极个别的暂停时间过长。当某次GC停顿太久, 就会影响程序的延迟指标. 如果延迟需求规定事务必须在 1,000 ms内完成, 那就不能容忍任何超过 1000毫秒的GC暂停。
- 老年代的使用量超过限制。如果老年代空间在 Full GC 之后仍然接近满值, 那么GC就成为了性能瓶颈, 可能是内存太小, 也可能存在内存泄漏。这种症状就会让GC开销暴涨。


As we can see, GC logs can give us very detailed information about what is going on inside the JVM in relation to garbage collection. However, for all but the most trivial applications, this results in vast amounts of data, which tend to be difficult to read and interpret by a human being.

我们可以看到,日志中显示了非常详细的GC信息。但除了最简单的Demo程序, 其他情况下都会生成大量的日志数据, 纯靠人工是很难进行阅读和理解的。





## GCViewer



One way to cope with the information flood in GC logs is to write custom parsers for GC log files to visualize the information. In most cases it would not be a reasonable decision due to the complexity of different GC algorithms that are producing the information. Instead, a good way would be to start with a solution that already exists: GCViewer.

可以编写自定义解析器, 来将庞大的GC日志文件解析为直观易读的图形信息。 但很多时候这并不是最好的解决方案, 因为各种GC算法的复杂性, 导致日志信息格式互不兼容。那么神器来了: [GCViewer](https://github.com/chewiebug/GCViewer)。


[GCViewer](https://github.com/chewiebug/GCViewer) is an open source tool for parsing and analyzing GC log files. The GitHub web page provides a full description of all the presented metrics. In the following we will review the most common way to use this tool.

[GCViewer](https://github.com/chewiebug/GCViewer) 是一款开源的GC日志分析工具。项目的 GitHub 主页对各个指标提供了完整的描述信息. 下面我们介绍最常用的指标。


The first step is to get useful garbage collection log files. They should reflect the usage scenario of your application that you are interested in during this performance optimization session. If your operational department complains that every Friday afternoon, the application slows down, and if you want to verify whether GC is the culprit, there is no point in analyzing GC logs from Monday morning.

第一步是获取GC日志文件。这些日志文件要能够反映系统在性能调优时的具体场景. 假若运营部门(operational department)反馈每周五下午,程序就运行缓慢, 不管GC是不是主要原因,从周一早晨的日志开始分析是没有多少意义的。


After you have received the log, you can feed it into GCViewer to see a result similar to the following:

收到日志文件之后, 可以用 GCViewer 进行分析,大致会看到类似下面的结果:


![](06_04_gcviewer-screenshot.png)


使用的命令行大致如下:

	java -jar gcviewer_1.3.4.jar gc.log


当然, 如果不想打开程序界面,也可以在后面加上其他参数,直接将分析结果输出到文件。

大致是这样: 

	java -jar gcviewer_1.3.4.jar gc.log summary.csv chart.png


点击下载: [gcviewer的jar包及使用示例](http://download.csdn.net/detail/renfufei/9753654) 。


The chart area is devoted to the visual representation of GC events. The information available includes the sizes of all memory pools and GC events. In the picture above, only two metrics are visualized: the total used heap is visualized with blue lines and individual GC pause durations with black bars underneath.

Chart 区域是对GC事件的图形化显示。展示的信息包括所有内存池的大小和GC事件。在上面的图片中,只有两个可视化指标: 蓝色线条表示堆内存的使用情况, 黑色的Bar则表示每次GC暂停时间的长短。


The first interesting thing that we can see from this picture is the fast growth of memory usage. In just around a minute the total heap consumption reaches almost the maximum heap available. This indicates a problem – almost all the heap is consumed and new allocations cannot proceed, triggering frequent full GC cycles. The application is either leaking memory or set to run with under-provisioned heap size.

从图中可以看到, 内存的使用量快速增长。仅仅一分钟左右就达到了堆内存的最大可用值.  几乎所有堆内存都被消耗, 新的内存分配不能顺利进行, 并引发频繁的 Full GC周期. 这说明程序可能存在内存泄露, 或者在启动时指定的内存空间不够。


The next problem visible in the charts is the frequency and duration of GC pauses. We can see that after the initial 30 seconds GC is almost constantly running, with the longest pauses exceeding 1.4 seconds.

从图中还可以看出, GC暂停的频率和持续时间。在最初的30秒之后, GC几乎不间断地运行,最长的暂停时间超过1.4秒。


On the right there is small panel with three tabs. Under the “Summary” tab the most interesting numbers are “Throughput” and “Number of GC pauses”, along with the “Number of full GC pauses”. Throughput shows the portion of the application run time that was devoted to running your application, as opposed to spending time in garbage collection.

在右边有三个选项卡。“**Summary**(摘要)” 中比较有用的是 “Throughput”(吞吐量百分比) 和 “Number of GC pauses”(GC暂停的次数), 以及“Number of full GC pauses”(Full GC 暂停的次数). 吞吐量显示了有多少CPU时间在执行程序, 剩下的就是垃圾收集所消耗的时间。


In our example we are facing throughput of 6.28%. This means that 93.72% of the CPU time was spent on GC. It is a clear symptom of the application suffering – instead of spending precious CPU cycles on actual work, it spends most of the time trying to get rid of the garbage.

示例中的吞吐量是 **6.28%**。这意味着有 **93.72%** 的CPU时间花在了GC上面. 很明显程序面临一个困境 —— 没有把宝贵的CPU时间用在实际工作上, 大部分的时间都在试图清理垃圾。


The next interesting tab is “Pause”:

下一个有意思的地方是“**Pause**”(暂停)选项卡:


![](06_05_gviewer-screenshot-pause.png)




The “Pause” tab exposes the totals, averages, minimum and maximum values of GC pauses, both as a grand total and minor/major pauses separately. Optimizing the application for low latency, this gives the first glimpse of whether or not you are facing pauses that are too long. Again, we can get confirmation that both the accumulated pause duration of 634.59 seconds and the total number of GC pauses of 3,938 is much too high, considering the total runtime of just over 11 minutes.

“Pause” 展示了GC暂停的总时间,平均值,最小值和最大值, 并且将 total 与minor/major 暂停分开统计。如果要优化程序的低延迟, 这可以让你一眼就能判断出暂停时间是否过长了。另外, 我们可以得出明确的信息: 累计的暂停时间是 `634.59` 秒, GC暂停的总次数为 3,938 次, 这在11分钟的总运行时间里确实是太高了。


More detailed information about GC events can be obtained from the “Event details” tab of the main area:

更详细的GC暂停汇总信息, 请查看主界面中的 “**Event details**” 标签:


![](06_06_gcviewer-screenshot-eventdetails.png)




Here you can see a summary of all the important GC events recorded in the logs: minor and major pauses and concurrent, not stop-the-world GC events. In our case, we can see an obvious “winner”, a situation in which full GC events are contributing the most to both throughput and latency, by again confirming the fact that the 3,928 full GC pauses took 634 seconds to complete.

从“**Event details**” 标签, 可以看到日志中所有重要的GC事件汇总: 普通GC停顿 和 Full GC 停顿次数, 以及并发执行数, 非 stop-the-world 事件等。此示例中, 可以看到有一个明显的地方, Full GC 暂停验证影响了吞吐量和延迟, 事实是: 3,928 次 Full GC, 暂停了634秒。


As seen from the example, GCViewer can quickly visualize symptoms that tell us whether or not the JVM is healthy in terms of misbehaving garbage collectors. As general guidelines, the following symptoms can quickly be revealed, visualizing the behavior of the GC:

可以看到, GCViewer 能用可视化的方式快速展现JVM中异常的GC行为。一般来说, 图像化的信息能迅速揭示以下症状:


- Low throughput. When the throughput of the application decreases and falls under a tolerable level, the total time that the application spends doing useful work gets reduced. What is “tolerable” depends on your application and its usage scenario. One rule of thumb says that any value below 90% should draw your attention and may require GC optimization.
- Excessively long individual GC pauses. Whenever an individual pause starts taking too long, the latency of the application starts to suffer. When the latency requirements require the transactions in the application to complete under 1,000 ms, for example, you cannot tolerate any GC pauses taking more than 1,000 ms.
- High heap consumption. Whenever the old generation remains close to being fully utilized even after several full GC cycles, you face a situation where the application is at its limits, either due to under-provisioning resources or due to a memory leak. This symptom always has a significant impact on throughput as well.

<br/>

- 低吞吐量。当应用的吞吐量下降到不能容忍的地步时, 有用的工作的总时间就大量减少. 具体有多大的 “容忍度”(tolerable) 取决于具体场景。按照经验, 低于 90% 的有效时间就值得警惕了, 可能真的需要好好优化下GC。
- 单次GC的暂停时间过长。只要有一次GC停顿时间过长,就会影响程序的延迟指标. 例如, 如果延迟需求指明要在 1000 ms以内完成交易, 那就不能容忍任何一次超过1000毫秒的GC暂停。
- 堆内存使用率过高。如果老年代空间在 Full GC 之后仍然全部占用, 那么程序性能就会大幅降低, 有可能是资源不足或者内存泄漏。这个症状会对吞吐量产生严重影响。


As a general comment – visualizing GC logs is definitely something we recommend. Instead of directly working with lengthy and complex GC logs, you get access to humanly understandable visualization of the very same information.

业界良心 —— 图形化展示的GC日志信息绝对是我们重磅推荐的。不用去直面冗长而又复杂的GC日志,通过易于理解的图形你可以得到同样的信息。



## 分析器(Profilers)



The next set of tools to introduce is [profilers](http://zeroturnaround.com/rebellabs/developer-productivity-report-2015-java-performance-survey-results/3/). As opposed to the tools introduced in previous sections, GC-related areas are a subset of the functionality that profilers offer. In this section we focus only on the GC-related functionality of profilers.

下面介绍分析器([profilers](http://zeroturnaround.com/rebellabs/developer-productivity-report-2015-java-performance-survey-results/3/), Oracle官方翻译是:抽样器)。相对于前面介绍的工具, 分析器只关心GC的一部分领域. 本节我们只关注分析器相关的GC功能。


The chapter starts with a warning – profilers as a tool category tend to be misunderstood and used in situations for which there are better alternatives. There are times when profilers truly shine, for example when detecting CPU hot spots in your application code. But for several other situations there are better alternatives.


首先警告 —— 分析器往往会被认为适合所有的场景。有时候分析器确实功勋卓著, 比如检测代码中的CPU热点时。但对某些情况不一定是个好方案。



This also applies to garbage collection tuning. When it comes to detecting whether or not you are suffering from GC-induced latency or throughput issues, you do not really need a profiler. The tools mentioned in previous chapters (jstat or raw/visualized GC logs) are quicker and cheaper ways of detecting whether or not you have anything to worry about in the first place. Especially when gathering data from production deployments, profilers should not be your tool of choice due to the introduced performance overhead.

对GC调优来说也是同样的。要检测是否因为GC而引起延迟或吞吐量问题时,并不需要使用分析器. 前面提到的工具( jstat 或 原生/可视化GC日志)都能更好更快地检测出是否需要关注GC. 特别是从生产环境收集数据时, 最好不要选择分析器, 因为性能开销实在是太大了。


But whenever you have verified you indeed need to optimize the impact GC has on your application, profilers do have an important role to play by exposing information about object creation. If you take a step back – GC pauses are triggered by objects not fitting into a particular memory pool. This can only happen when you create objects. And all profilers are capable of tracking object allocations via allocation profiling, **giving you information about what actually resides in the memory along with the allocation traces**.

如果确定需要对GC进行优化, 那么分析器就可以发挥重要的作用, 让 Object 的创建信息一目了然. 再往前看, 造成大量GC暂停的原因不在某个特定内存池中。那就只能发生创建对象的时候. 所有的分析器都能够跟踪对象分配(via allocation profiling), 根据内存分配的轨迹, 让你知道**实际驻留在内存中的是哪些对象**。


Allocation profiling gives you information about the places where your application creates the majority of the objects. Exposing the top memory consuming objects and the threads in your application that produce the largest number of objects is the benefit you should be after when using profilers for GC tuning.

分配分析能定位到哪个地方创建了大量的对象. 使用分析器辅助进行GC调优的好处是, 能确定是什么类型最占用内存, 以及是哪些线程生成了最多的对象。


In the following sections we will see three different allocation profilers in action: hprof, JVisualVM and AProf. There are plenty more different profilers out there to choose from, both commercial and free solutions, but the functionality and benefits of each are similar to the ones discussed in the following sections.

我们将在实例中介绍三种不同的分配分析器: **`hprof`**, **`JVisualV`M** 和 **`AProf`**。实际上还有很多分析器可供选择, 包括商业的和免费的, 但其功能和优点都类似于下面讨论的这些。



### hprof


Bundled with the JDK distribution is [hprof profiler](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html). As it is present in all environments, this is the first profiler to be considered when harvesting information.

[hprof 分析器](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html)内置于JDK之中。在各种环境都可以使用, 一般优先使用这款工具。


To run your application with hprof, you need to modify the startup scripts of the JVM similarly to the following example:

要让 hprof 和程序一起运行, 需要修改启动脚本, 类似这样:


	java -agentlib:hprof=heap=sites com.yourcompany.YourApplication



On application exit, it will dump the allocation information to a java.hprof.txt file to the working directory. Open the file with a text editor of your choice and search for the phrase “SITES BEGIN” giving you information similar to the following:

在程序退出时,会将分配信息dump(转储)到工作目录下的 java.hprof.txt 文件。使用文本编辑器打开, 并搜索 “**SITES BEGIN**” 关键字, 可以看到:


	SITES BEGIN (ordered by live bytes) Tue Dec  8 11:16:15 2015
	          percent          live          alloc'ed  stack class
	 rank   self  accum     bytes objs     bytes  objs trace name
	    1  64.43% 4.43%   8370336 20121  27513408 66138 302116 int[]
	    2  3.26% 88.49%    482976 20124   1587696 66154 302104 java.util.ArrayList
	    3  1.76% 88.74%    241704 20121   1587312 66138 302115 eu.plumbr.demo.largeheap.ClonableClass0006
	    ... 部分省略 ...
	
	SITES END




From the above, we can see the allocations ranked by the number of objects created per allocation. The first line shows that 64.43% of all objects were integer arrays (int[]) created at the site identified by the number 302116. Searching the file for “TRACE 302116” gives us the following information:

从以上片段可以看到, allocations 是根据每次创建的对象数量来排序的。第一行显示所有对象中有 **64.43%** 的对象是整型数组(`int[]`), 在标识为数字 302116 的位置创建。搜索 “**TRACE 302116**” 可以看到:


	TRACE 302116:	
		eu.plumbr.demo.largeheap.ClonableClass0006.<init>(GeneratorClass.java:11)
		sun.reflect.GeneratedConstructorAccessor7.newInstance(<Unknown Source>:Unknown line)
		sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		java.lang.reflect.Constructor.newInstance(Constructor.java:422)




Now, knowing that 64.43% of the objects were allocated as integer arrays in the ClonableClass0006 constructor, in the line 11, you can proceed with optimizing the code to reduce the burden on GC.

现在, 知道了 64.43% 的对象被分配为整数数组, 在 ClonableClass0006 类的构造函数中, 第11行, 那么就可以优化代码, 以减少GC的压力。



### Java VisualVM


JVisualVM makes a second appearance in this chapter. In the first section we ruled JVisualVM out from the list of tools monitoring your JVM for GC behavior, but in this section we will demonstrate its strengths in allocation profiling.

这是本章第二次介绍 JVisualVM 了。在第一部分, 在监控 JVM 的GC行为工具中介绍了 JVisualVM , 本节将介绍其在分配分析上的优点。


Attaching JVisualVM to your JVM is done via GUI where you can attach the profiler to a running JVM. After attaching the profiler:

JVisualVM 通过GUI方式连接到正在运行的JVM。 连接上profiler以后 :


1. Open the “Profiler” tab and make sure that under the “Settings” checkbox you have enabled “Record allocations stack traces”.
2. Click the “Memory” button to start memory profiling.
3. Let your application run for some time to gather enough information about object allocations.
4. Click the “Snapshot” button. This will give you a snapshot of the profiling information gathered.

<br/>

1. 打开 “工具” --> “选项” 菜单, 点击 **性能分析(Profiler)** 标签, 新增配置, 选择 Profiler 内存, 确保勾选了 “Record allocations stack traces”(记录分配栈跟踪)。
2. 勾选 “Settings”(设置) 复选框, 在内存设置标签下,修改预设配置。
3. 点击 “Memory”(内存) 按钮开始进行内存分析。
4. 让程序运行一段时间,以收集关于对象分配的足够信息。
5. 单击下面一点的 “Snapshot”(快照) 按钮。可以取得收集到的信息快照。


![](06_07_01_trace.png)


After completing the steps above, you are exposed to information similar to the following:

完成上面的步骤后, 可以得到类似这样的信息:


![](06_07_jvisualvm-top-objects.png)




From the above, we can see the allocations ranked by the number of objects created per class. In the very first line we can see that the vast majority of all objects allocated were int[] arrays. By right-clicking the row you can access all stack traces where those objects were allocated:

上图是按照每个类的对象创建数量来排序的。从第一行可以看到,分配的最多的对象是 `int[]` 数组. 右键单击这行就可以看到这些对象都是在哪个地方分配的:


![](06_08_jvisualvm-allocation-traces.png)




Compared to hprof, JVisualVM makes the information a bit easier to process – for example in the screenshot above you can see that all allocation traces for the int[] are exposed in single view, so you can escape the potentially repetitive process of matching different allocation traces.

与 `hprof` 相比, JVisualVM 更加容易 —— 比如在上面的截图中, 在一个地方就可以看到所有的`int[]` 分配信息, 所以多次在同一处代码进行分配时就很容易看到了。





### AProf


Last, but not least, is AProf by Devexperts. AProf is a memory allocation profiler packaged as a Java agent.

第三款, 同时也是最重要的工具,是由 Devexperts 开发的 **[AProf](https://code.devexperts.com/display/AProf/About+Aprof)**。AProf 也是打包为 Java agent 的内存分配分析器。


To run your application with AProf, you need to modify the startup scripts of the JVM similarly to the following example:

用AProf 来分析应用程序, 需要修改JVM的启动脚本,类似这样:


	java -javaagent:/path-to/aprof.jar com.yourcompany.YourApplication



Now, after restarting the application, an aprof.txt file will be created to the working directory. The file is updated once every minute and contains information similar to the following:

重启应用程序以后, 工作目录下会生成一个 aprof.txt 文件。这个文件每分钟更新一次, 包含这样的信息:


	========================================================================================================================
	TOTAL allocation dump for 91,289 ms (0h01m31s)
	Allocated 1,769,670,584 bytes in 24,868,088 objects of 425 classes in 2,127 locations
	========================================================================================================================
	
	Top allocation-inducing locations with the data types allocated from them
	------------------------------------------------------------------------------------------------------------------------
	eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 1,423,675,776 (80.44%) bytes in 17,113,721 (68.81%) objects (avg size 83 bytes)
		int[]: 711,322,976 (40.19%) bytes in 1,709,911 (6.87%) objects (avg size 416 bytes)
		char[]: 369,550,816 (20.88%) bytes in 5,132,759 (20.63%) objects (avg size 72 bytes)
		java.lang.reflect.Constructor: 136,800,000 (7.73%) bytes in 1,710,000 (6.87%) objects (avg size 80 bytes)
		java.lang.Object[]: 41,079,872 (2.32%) bytes in 1,710,712 (6.87%) objects (avg size 24 bytes)
		java.lang.String: 41,063,496 (2.32%) bytes in 1,710,979 (6.88%) objects (avg size 24 bytes)
		java.util.ArrayList: 41,050,680 (2.31%) bytes in 1,710,445 (6.87%) objects (avg size 24 bytes)
	          ... cut for brevity ... 




In the output above allocations are ordered by size. From the above you can see right away that 80.44% of the bytes and 68.81% of the objects were allocated in the ManyTargetsGarbageProducer.newRandomClassObject() method. Out of these, int[] arrays took the most space with 40.19% of total memory consumption.

上面的输出是按照 size 进行排序的。可以看出, `80.44%` 的 bytes 和 68.81% 的 objects 是在 `ManyTargetsGarbageProducer.newRandomClassObject()` 方法中分配的。 其中, **int[]** 数组消耗了最多的内存, 占总量的 40.19%。


Scrolling further down the file, you will discover a block with allocation traces, also ordered by allocation sizes:

继续往下看, 你会发现 allocation traces(分配痕迹)相关的内容, 也是根据 allocation size 来排序的:


	Top allocated data types with reverse location traces
	------------------------------------------------------------------------------------------------------------------------
	int[]: 725,306,304 (40.98%) bytes in 1,954,234 (7.85%) objects (avg size 371 bytes)
		eu.plumbr.demo.largeheap.ClonableClass0006.: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
			java.lang.reflect.Constructor.newInstance: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
				eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 38,357,280 (2.16%) bytes in 92,205 (0.37%) objects (avg size 416 bytes)
				java.lang.reflect.Constructor.newInstance: 416 (0.00%) bytes in 1 (0.00%) objects (avg size 416 bytes)
	... cut for brevity ... 




From the above we can see the allocations for int[] arrays, again zooming in to the ClonableClass0006 constructor where these arrays were created.

可以看到, `int[]` 数组的分配, 在 ClonableClass0006 构造函数中继续增大。


So, like other solutions, AProf exposed information about allocation size and locations, making it possible to zoom in to the most memory-hungry parts of your application. However, in our opinion AProf is the most useful allocation profiler, as it focuses on just one thing and does it extremely well. In addition, this open-source tool is free and has the least overhead compared to alternatives.

和其他工具一样, AProf 揭露了 分配大小以及位置信息(allocation size and locations), 从而能够快速找到最耗内存的部分。在我们看来, **AProf** 是最有用的分配分析器, 因为它只专注于内存分配, 而且做的非常棒。 当然, 此工具是开源免费的, 资源开销也是最小的。




原文链接:  [GC Tuning: Tooling](https://plumbr.eu/handbook/gc-tuning-measuring)




<div style="page-break-after : always;"> </div>


