---
title: JVM 内存分析工具 MAT 的深度讲解与实践——入门篇(转载)
abbrlink: 73d10e8f
categories:
  - JVM 内存分析
date: 2021-08-20 15:06:50
tags:
  - JVM
  - MAT
cover: https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2021/08/20/20210820152607.png
reprint: true
---

> 原文作者：https://juejin.cn/user/1275089220539869
> 原文作者公众号： **Q的博客。**
> 原文链接：https://juejin.cn/post/6911624328472133646

JVM 内存分析往往由团队较资深的同学来做，本系列通过3篇文章，深度解析并帮助读者全面深度掌握 MAT 的使用方法。即使没有 JVM 内存分析的实践经验，也能快速成为内存分析高手！

**本系列共三篇文章如下，** 本文是第一篇入门篇：

*   **《JVM 内存分析工具 MAT 的深度讲解与实践——入门篇》** 介绍 MAT 产品功能、基础概念、与其他工具对比、Quick Start 指南。
*   **《JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇》** 展开并详细介绍 MAT 的核心功能，并在具体实战场景下讲解帮大家加深体会。
*   **《JVM 内存分析工具 MAT 的深度讲解与实践——高阶篇》** 总结复杂内存问题的系统性分析方法，并通过一个综合案例提升大家的实战能力。

<!--more-->

* * *

MAT（全名：Memory Analyzer Tool），是一款快速便捷且功能强大丰富的 JVM 堆内存离线分析工具。其通过展现 JVM 异常时所记录的运行时堆转储快照（Heap dump）状态（正常运行时也可以做堆转储分析），帮助定位内存泄漏问题或优化大内存消耗逻辑。

1.1 MAT 使用场景及主要解决问题
-------------------

场景一：内存溢出，JVM堆区或方法区放不下存活及待申请的对象。如：高峰期系统出现 OOM（Out of Memory）异常，需定位内存瓶颈点来指导优化。

场景二：内存泄漏，不会再使用的对象无法被垃圾回收器回收。如：系统运行一段时间后出现 Full GC，甚至周期性 OOM 后需人工重启解决。

场景三：内存占用高。如：系统频繁 GC ，需定位影响服务实时性、稳定性、吞吐能力的原因。

1.2 基础概念
--------

### 1.2.1 Heap Dump

Heap Dump 是 Java 进程堆内存在一个时间点的快照，支持 HPROF 及 DTFJ 格式，前者由 Oracle 系列 JVM 生成，后者是 IBM 系列 JVM 生成。其内容主要包含以下几类：

*   所有对象的实例信息：对象所属类名、基础类型和引用类型的属性等。
    
*   所有类信息：类加载器、类名、继承关系、静态属性等。
    
*   GC Root：GC Root 代表通过可达性分析来判定 JVM 对象是否存活的起始集合。JVM 采用追踪式垃圾回收（Tracing GC）模式，**从所有 GC Roots 出发通过引用关系可以关联的对象**就是存活的（且不可回收），其余的不可达的对象（Unreachable object：如果无法从 GC Root 找到一条引用路径能到达某对象，则该对象为Unreachable object）可以回收。
    
*   线程栈及局部变量：快照生成时刻的所有线程的线程栈帧，以及每个线程栈的局部变量。
    

### 1.2.2 Shallow Heap

Shallow Heap 代表一个**对象结构自身**所占用的内存大小，不包括其属性引用对象所占的内存。如 java.util.ArrayList 对象的 Shallow Heap 包含8字节的对象头、8字节的对象数组属性 elementData 引用 、 4字节的 size 属性、4字节的 modCount 属性（从 AbstractList 继承及对象头占用内存大小），有的对象可能需要加对齐填充但 ArrayList 自身已对齐不需补充，注意不包含 elementData 具体数据占用的内存大小。

### 1.2.3 Retained Set

一个对象的 Retained Set，指的是该对象被 GC 回收后，所有能被回收的对象集合（如下图所示，G的 Retain Set 只有 G 并不包含 H，原因是虽然 H 也被 G 引用，但由于 H 也被 F 引用 ，G 被垃圾回收时无法释放 H）；另外，当该对象无法被 GC 回收，则其 Retained set 也必然无法被 GC 回收。

### 1.2.4 Retained Heap

Retained Heap 是一个对象被 GC 回收后，可释放的内存大小，等于释放对象的 Retained Heap 中所有对象的 Shallow Heap 的和（如下图所示，E 的 Retain Heap 就是 G 与 E 的 Shallow Heap 总和，同理不包含 H）。

### 1.2.5 Dominator tree

如果所有指向对象 Y 的路径都经过对象 X，则 X 支配（dominate） Y（如下图中，C、D 均支配 F，但 G 并不支配 H）。Dominator tree 是根据对象引用及支配关系生成的整体树状图，支配树清晰描述了对象间的依赖关系，下图左的 Dominator tree 如下图右下方支配树示意图所示。支配关系还有如下关系：

*   Dominator tree 中任一节点的子树就是被该节点支配的节点集合，也就是其 Retain Set。
    
*   如果 X 直接支配 Y，则 X 的所有支配节点均支配 Y。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/107a97507c554800a93a24c4ac4ec279~tplv-k3u1fbpfcp-watermark.image)
    

### 1.2.6 OQL

OQL 是类似于 SQL 的 MAT 专用统一查询语言，可以根据复杂的查询条件对 dump 文件中的类或者对象等数据进行查询筛选。

### 1.2.7 references

outgoing references、incoming references 可以直击对象间依赖关系，MAT 也提供了链式快速操作。

*   outgoing references：对象引用的外部对象（注意不包含对象的基本类型属性。基本属性内容可在 inspector 查看）。
    
*   incoming references：直接引用了当前对象的对象，每个对象的 incoming references 可能有 0 到多个。
    

2.1 MAT 功能概述
------------

_注：MAT 的产品能力非常丰富，本文简要总结产品特性帮大家了解全貌，在下一篇文章《JVM 内存分析实战进阶篇——核心功能及应用场景》中，会详细展开介绍各项核心功能的场景、案例、最佳实践等。_

MAT 的工作原理是对 dump 文件建立多种索引，并基于索引来实现 \[1\]内存分布、\[2\]对象间依赖（如实体对象引用关系、线程引用关系、ClassLoader引用关系等）、\[3\]对象状态（内存占用量、字段属性值等）、\[4\]条件检索（OQL、正则匹配查询等）这四大核心功能，并通过可视化展现辅助 Developer 精细化了解 JVM 堆内存全貌。

### 2.1.1 内存分布

*   全局概览信息：堆内存大小、对象个数、类的个数、类加载器的个数、GC root 个数、线程概况等全局统计信息。
    
*   Dominator tree：按对象的 Retain Heap 排序，也支持按多个维度聚类统计，最常用的功能之一。
    
*   Histogram：罗列每个类实例的内存占比，包括自身内存占用量（Shallow Heap）及支配对象的内存占用量（Retain Heap），支持按 package、class loader、super class、class 聚类统计，最常用的功能之一。
    
*   Leak Suspects：直击引用链条上占用内存较多的可疑对象，可解决一些基础问题，但复杂的问题往往帮助有限。
    
*   Top Consumers：展现哪些类、哪些 class loader、哪些 package 占用最高比例的内存。
    

### 2.1.2 对象间依赖

*   References：提供对象的外部引用关系、被引用关系。通过任一对象的直接引用及间接引用详情（主要是属性值及内存占用），进而提供完善的依赖链路详情。
    
*   Dominator tree：支持按对象的 Retain Heap 排序，并提供详细的支配关系，结合 references 可以实现大对象快速关联分析；
    
*   Thread overview：展现转储 dump 文件时线程栈帧等详细状态，也提供各线程的 Retain Heap 等关联内存信息。
    
*   Path To GC Roots：提供任一对象到GC Root的链路详情，帮助了解不能被 GC 回收的原因。
    

### 2.1.3 对象状态

*   最核心的是通过 inspector 面板提供对象的属性信息、类继承关系信息等数据，协助分析内存占用高与业务逻辑的关系。
    
*   集合状态的检测，如：通过 ArrayList 或数组的填充率定位空集合空数组造成的内存浪费、通过 HashMap 冲突率判定 hash 策略是否合理等。
    

### 2.1.4 按条件检索对象

*   OQL：提供一种类似于SQL的对象（类）级别统一结构化查询语言。如：查找 size＝0 且未使用过的 ArrayList： select * from java.util.ArrayList where size=0 and modCount=0；查找所有的String的length属性的： select s.length from instanceof String s。
    
*   内存分布及对象间依赖的众多功能，均支持按字符串检索、按正则检索等操作。
    
*   按虚拟内存地址寻址，根据对象的十六进制地址查找对象。
    

此外，为了便于记忆与回顾，整理了如下脑图： ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e4caeb5f3d541fe8b5f6968dc9cb2ef~tplv-k3u1fbpfcp-watermark.image)

2.2 常见内存分析工具对比
--------------

下图中 Y 表示支持，N 表示不支持，时间截至发稿前。

| 产品功能 | MAT | JProfiler | Visual VM | jhat | jmap | hprof |
| --- | --- | --- | --- | --- | --- | --- |
| 对象关联分析、深浅堆、GC ROOT、内存泄漏检测、线程分析、提供自定义程序扩展扩展 | Y | N | N | N | N | N |
| 离线全局分析 | Y | N | Y | Y | N | N |
| 内存实时分配情况 | N | Y | Y | Y | Y | Y |
| OQL | Y | N | Y | N | N | N |
| 内存分配堆栈、热点比例 | N | Y | N | N | N | N |
| 堆外内存分析 | N | N | N | N | N | N |

_注 1：Dump 文件包含快照被转储时刻的 Java 对象 在堆内存中的分布情况，但快照只是瞬间的记录，所以不包含对象在何时、在哪个方法中被分配这类信息。_

_注 2：一般堆外内存溢出排查可结合 gperftools 与 btrace 排查，此类文章较多不展开介绍。_

3.1 Quick Start
---------------

_注：Quick Start 文章较多，本文着重介绍安装流程及使用技巧。_

1、安装 MAT：戳【[下载链接](https://link.juejin.cn/?target=http%3A%2F%2Fwww.eclipse.org%2Fmat%2Fdownloads.php "http://www.eclipse.org/mat/downloads.php")】；也可直接集成到 Eclipse IDE中（路径：Eclipse → Help → Eclipse Marketplace → 搜 “MAT”）。

2、调节 MAT 堆内存大小：MAT 分析时也作为 Java 进程运行，如果有足够的内存，建议至少分配 dump 文件大小*1.2 倍的内存给 MAT，这样分析速度会比较快。方式是修改MemoryAnalyer.ini文件，调整Xmx参数（Windows 可用搜索神器 everything 软件查找并修改、MAC OS 一般在 /Applications/mat.app/Contents/Eclipse/MemoryAnalyzer.ini，如找不到可用 Alfred 软件查询修改）。

3、获取堆快照 dump 文件（堆转储需要先执行 Full GC，线上服务使用时请注意影响），一般用三种方式：

*   使用 JDK 提供的 jmap 工具，命令是 jmap -dump:format=b,file=文件名 进程号。当进程接近僵死时，可以添加 -F 参数强制转储：jmap -F -dump:format=b,file=文件名 进程号。
    
*   本地运行的 Java 进程，直接在 MAT 使用 File → accquire heap dump 功能获取。
    
*   启动 Java 进程时配置JVM参数：-XX:-HeapDumpOnOutOfMemoryError，当发生 OOM 时无需人工干预会自动生成 dump文件。指定目录用 -XX:HeapDumpPath=文件路径 来设置。
    

4、分析 dump 文件：路径是 File → Open Heap Dump ，然后 MAT 会建立索引并分析，dump 文件较大时耗时会很长。分析后 dump 文件所在目录会有后缀为 index 的索引文件，也会有包含 HTML 格式的后缀为 zip 的文件。

5、完成索引计算后，MAT 呈现概要视图（Overview），包含三个部分：

*   全局概览信息，堆内存大小、类数量、实例数量、Class Loader数量。
    
*   Unreachable Object Histogram，展现转储快照时可被回收的对象信息（一般不需要关注，除非 GC 频繁影响实时性的场景分析才用到）
    
*   Biggest Objects by Retained Size，展现经过统计过的哪几个实例所关联的对象占内存总和较高，以及具体占用的内存大小，一般相关代码比较简单情况下，往往可以直接分析具体的引用关系异常，如内存泄漏等。此外也包含了最大对象和链接支持继续深入分析。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4873700bf5fc4747afa900439a71e1d0~tplv-k3u1fbpfcp-watermark.image)
    

6、如果代码比较复杂，需要继续使用 MAT 各种工具并结合业务代码进一步分析内存异常的原因。最常用的几项如下（具体案例、场景、使用方式在《JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇》详细介绍）：

*   查看堆整体情况的：Histogram、Dominator tree、Thread details等（各功能入口整理如下） ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53fa4124ff9e4d8f9aadd407ac41a784~tplv-k3u1fbpfcp-watermark.image)
    
*   MAT 分析过的 Top Consumers 、Leak Suspects等 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f203c536a2145048444af0f0b60d487~tplv-k3u1fbpfcp-watermark.image)
    

3.2 使用技巧及注意事项
-------------

1、注意对运行进程的性能影响：Heap dump 时会先进行 Full GC，另外为保证对象数据视图一致，需要在安全点 Stop The World 暂停响应，线上服务进行务必注意性能影响。可以采取以下技巧减少影响：

*   先禁用入口流量，再执行 dump 动作。
    
*   选择影响较小时 dump 内存。
    
*   使用脚本捕获指定事件时 dump 内存。
    

2、Dump 文件及建立的索引文件可能较大，如果开发机配置不足无法分析，可在服务器先执行分析后，基于分析后的索引文件直接查看结果，另外也需要注意磁盘占用问题：

*   大文件分析方法：一般 dump 文件不高于分析机主存 1.2 倍可直接在开发机分析；**若 dump 文件过大，可以使用 MAT 提供的脚本在配置高的高配机器先建立索引再直接展现索引分析结果**（一般是 Linux 机器，可以使用 MAT 提供的脚本：./ParseHeapDump.sh $HEAPDUMP，堆信息有 unreachable 标记的垃圾对象，在 dump 时也保存了下来，默认不分析此部分数据，如需要在启动脚本 ParseHeapDump.sh 中加入：-keep\_unreachable\_objects）。
    
*   如果不关注堆中不可达对象，使用“live”参数可以减小文件大小，命令是 jmap -dump:live,format=b,file=<dumpfile> <pid>
    
*   Dump 前主动手动执行一次 FULL GC ，去除无效对象进一步减少 dump 堆转储及建立索引的时间。
    
*   Dump文件巨大，建立索引后发现主视图中对象占用内存均较小，这是因为绝大部分对象未被 GC Roots 引用可释放。
    
*   Dump 时注意指定到空间较大的磁盘位置，避免打满分区影响服务。
    
*   建立 dump 索引机器的磁盘空间需要足够大，一般至少是 dump 文件的两倍，因为生成的中间索引文件也较大，如下图： ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e485cfbabe8b47df88ec66f663734505~tplv-k3u1fbpfcp-watermark.image)
    

3、其他

*   JDK 版本问题：如遇“VMVersionMismatchException”，使用启动目标进程的 JDK 版本即可。
    
*   部分核心功能主界面未展现，问题足够复杂时需打开，如 MAT 默认不打开 inspector，如需根据对象数据值做业务分析，建议打开该视图。
    
*   配置了 HeapDumpOnOutOfMemoryError 参数，但 OutOfMemoryError 时但没有自动生成 dump 文件，可能原因有三个：
    
    *   应用程序自行创建并抛出 OutOfMemoryError
    *   进程的其他资源（如线程）已用尽
    *   C 代码（如 JVM 源码）中堆耗尽，这种可能由于不同的原因而出现，例如在交换空间不足的情况下，进程限制用尽或仅地址空间的限制，此时 dump 文件分析并无实质性帮助。

总结展望：至此本文讲解了 MAT 实践必备的初级内容，在下一篇《JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇》会展开并详细介绍 MAT 丰富的核心功能，每个功能点讲解都会从具体场景聊起，帮助大家在实战场景下加深体会实现进阶。

*   MAT官网：[help.eclipse.org/2020-09/ind…](https://link.juejin.cn/?target=https%3A%2F%2Fhelp.eclipse.org%2F2020-09%2Findex.jsp%3Ftopic%3D%2Forg.eclipse.mat.ui.help%2Fwelcome.html "https://help.eclipse.org/2020-09/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html")
