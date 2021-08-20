---
title: JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇(转载)
abbrlink: 5c59cf1f
categories:
  - JVM 内存分析
date: 2021-08-20 15:35:55
tags:
  - JVM
  - MAT
cover: https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2021/08/20/20210820152607.png
reprint: true
---

> 原文作者：https://juejin.cn/user/1275089220539869
> 原文作者公众号： **Q的博客。**
> 原文链接：https://juejin.cn/post/6911624328472133646


**本系列共三篇文章，** 本文是系列第2篇——进阶篇，详细讲解 MAT 各种工具的**核心功能、用法、适用场景**，并在具体实战场景下讲解帮大家学习如何针对各类内存问题。

*   《[JVM 内存分析工具 MAT 的深度讲解与实践——入门篇](https://juejin.cn/post/6908665391136899079 "https://juejin.cn/post/6908665391136899079")》 介绍 MAT 产品功能、基础概念、与其他工具对比、Quick Start 指南。
*   **《JVM 内存分析工具 MAT 的深度讲解与实践——进阶篇》** 展开并详细讲解 MAT 各种工具的核心功能、用法、场景，并在具体实战场景下讲解帮大家加深体会。
*   《JVM 内存分析工具 MAT 的深度讲解与实践——高阶篇》 总结复杂内存问题的系统性分析方法，并通过一个综合案例提升大家的实战能力。

<!--more-->

* * *

熟练掌握 MAT 是 Java 高手的必备能力，但实践时大家往往需面对众多功能，眼花缭乱不知如何下手，小编也没有找到一篇完善的教学素材，所以整理本文帮大家系统掌握 MAT 分析工具。

本文详细讲解 MAT 众多内存分析工具功能，这些功能组合使用异常强大，熟练使用几乎可以解决所有的堆内存离线分析的问题。我们将功能划分为4类：内存分布详情、对象间依赖、对象状态详情、按条件检索。每大类有多个功能点，本文会逐一讲解各功能的场景及用法。此外，添加了原创或引用案例加强理解和掌握。

注：在该系列开篇文章《[JVM 内存分析工具 MAT 的深度讲解与实践——入门篇](https://juejin.cn/post/6908665391136899079 "https://juejin.cn/post/6908665391136899079")》中介绍了 MAT 的使用场景及安装方法，不熟悉 MAT 的读者建议先阅读上文并安装，本文案例很容易在本地实践。另外，上文中产品介绍部分顺序也对应本文功能讲解的组织，如下图： ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8bb5e21bfb5a4a4d855e5e0ee80206d7~tplv-k3u1fbpfcp-watermark.image)
 为减少对眼花缭乱的菜单的迷茫，可以通过下图先整体熟悉下各功能使用入口，后续都会讲到。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/822515f03c704e61ae8967048a36b4b3~tplv-k3u1fbpfcp-watermark.image)

2.1 全局信息概览
----------

**功能**：展现堆内存大小、对象数量、class 数量、class loader 数量、GC Root 数量、环境变量、线程概况等全局统计信息。

**使用入口**：MAT 主界面 → Heap Dump Overview。

**举例**：下面是对象数量、class loader 数量、GC Root 数量，可以看出 class loader 存在异常。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdfda3beda914ae288b96f261d0e58ca~tplv-k3u1fbpfcp-watermark.image)

**举例**：下图是线程概况，可以查看每个线程名、线程的 Retained Heap、daemon 属性等。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/542dac8375904537b24c22c7d9f1ff73~tplv-k3u1fbpfcp-watermark.image)

**使用场景** 全局概览呈现全局统计信息，重点查看整体是否有异常数据，所以有效信息有限，下面几种场景有一定帮助：

*   方法区溢出时（Java 8后不使用方法区，对应堆溢出），查看 class 数量异常多，可以考虑是否为动态代理类异常载入过多或类被反复重复加载。
*   方法区溢出时，查看 class loader 数量过多，可以考虑是否为自定义 class loader 被异常循环使用。
*   GC Root 过多，可以查看 GC Root 分布，理论上这种情况极少会遇到，笔者只在 JNI 使用一个存在 BUG 的库时遇到过。
*   线程数过多，一般是频繁创建线程但无法执行结束，从概览可以了解异常表象，具体原因可以参考本文线程分析部分内容，此处不展开。

2.2 Dominator tree
------------------

_**注：笔者使用频率的 Top1，是高效分析 Dump 必看的功能。** _

**功能**

*   展现对象的支配关系图，并给出对象支配内存的大小（支配内存等同于 Retained Heap，即其被 GC 回收可释放的内存大小）
*   支持排序、支持按 package、class loader、super class、class 聚类统计

**使用入口**：全局支配树： MAT 主界面 → Dominator tree。

**举例：**  下图中通过查看 Dominator tree，了解到内存主要是由 ThreadAndListHolder-thread 及 main 两个线程支配（后面第2.6节会给出整体案例）。

*   ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2398433f51d44bc193f5ff52b7ae3359~tplv-k3u1fbpfcp-watermark.image)
    

**使用场景**

*   开始 Dump 分析时，首先应使用 Dominator tree 了解各支配树起点对象所支配内存的大小，进而了解哪几个起点对象是 GC 无法释放大内存的原因。
    
*   当个别对象支配树的 Retained Heap 很大存在明显倾斜时，可以重点分析占比高的对象支配关系，展开子树进一步定位到问题根因，如下图中可看出最终是 SameContentWrapperContainer 对象持有的 ArrayList 过大。
    
    ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c4dd3d11eb4424bb034507e5c18afad~tplv-k3u1fbpfcp-watermark.image)
    
*   在 Dominator tree 中展开树状图，可以查看支配关系路径（与 outgoing reference 的区别是：如果 X 支配 Y，则 X 释放后 Y必然可释放；如果仅仅是 X 引用 Y，可能仍有其他对象引用 Y，X 释放后 Y 仍不能释放，所以 Dominator tree 去除了 incoming reference 中大量的冗余信息）。
    
*   有些情况下可能并没有支配起点对象的 Retained Heap 占用很大内存（比如 class X 有100个对象，每个对象的 Retained Heap 是10M，则 class X 所有对象实际支配的内存是 1G，但可能 Dominator tree 的前20个都是其他class 的对象），这时可以按 class、package、class loader 做聚合，进而定位目标。
    
    *   下图中各 GC Roots 所支配的内存均不大，这时需要聚合定位爆发点。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/565744afcf404deb9fd14ea7282cbc1d~tplv-k3u1fbpfcp-watermark.image)
        
    *   在 Dominator tree 展现后按 class 聚合，如下图： ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67954d39202b407e8c2e36fa471cdcf4~tplv-k3u1fbpfcp-watermark.image)
        
    *   可以定位到是 SomeEntry 对象支配内存较多，然后结合代码进一步分析具体原因。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d6f2ec751aa484ab8e5586465ab4f88~tplv-k3u1fbpfcp-watermark.image)
        
*   在一些操作后定位到异常持有 Retained Heap 对象后（如从代码看对象应该被回收），可以获取对象的直接支配者，操作方式如下。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b26e5c1700924cadbaae240c92c50547~tplv-k3u1fbpfcp-watermark.image)
    

2.3 Histogram 直方图
-----------------

_**注：笔者使用频率 Top2**_

**功能**

*   罗列每个类实例的数量、类实例累计内存占比，包括自身内存占用量（Shallow Heap）及支配对象的内存占用量（Retain Heap）。
*   支持按对象数量、Retained Heap、Shallow Heap（默认排序）等指标排序；支持按正则过滤；支持按 package、class loader、super class、class 聚类统计，

**使用入口**：MAT 主界面 → Histogram；注意 Histogram 默认不展现 Retained Heap，可以使用计算器图标计算，如下图所示。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95bd5af74fb640c5bef9eb2deacbbf44~tplv-k3u1fbpfcp-watermark.image)

**使用场景**

*   有些情况 Dominator tree 无法展现出热点对象（上文提到 Dominator tree 支配内存排名前20的占比均不高，或者按 class 聚合也无明显热点对象，此时 Dominator tree 很难做关联分析判断哪类对象占比高），这时可以使用 Histogram 查看所有对象所属类的分布，快速定位占据 Retained Heap 大头的类。
*   使用技巧

2.4 Leak Suspects
-----------------

**功能**：具备自动检测内存泄漏功能，罗列可能存在内存泄漏的问题点。

**使用入口**：一般当存在明显的内存泄漏时，分析完Dump文件后就会展现，也可以如下图在 MAT 主页 → Leak Suspects。

**使用场景**：需要查看引用链条上占用内存较多的可疑对象。这个功能可解决一些基础问题，但复杂的问题往往帮助有限。

**举例**

*   下图中 Leak Suspects 视图展现了两个线程支配了绝大部分内存。
    
    ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4cdfc1b12ac47cd8ffb739b68a77d0b~tplv-k3u1fbpfcp-watermark.image)
    
*   下图是点击上图中 Keywords 中 "Details" ,获取实例到 GC Root 的最短路径、dominator 路径的细信息。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a19c8ccf2ed44a0ae4150884665a0f6~tplv-k3u1fbpfcp-watermark.image)
    

2.5 Top Consumers
-----------------

**功能**：最大对象报告，可以展现哪些类、哪些 class loader、哪些 package 占用最高比例的内存，其功能 Histogram 及 Dominator tree 也都支持。

**使用场景**：应用程序发生内存泄漏时，查看哪些泄漏的对象通常在 Dump 快照中会占很大的比重。因此，对简单的问题具有较高的价值。

2.6 综合案例一
---------

**使用工具项**：Heap dump overview、Dominator tree、Histogram、Class Loader Explorer（见3.4节）、incoming references（见3.1节）

**程序代码**

```java
package com.q.mat;

import java.util.*;
import org.objectweb.asm.*;

public class ClassLoaderOOMOps extends ClassLoader implements Opcodes {

    public static void main(final String args[]) throws Exception {
        new ThreadAndListHolder(); 

        List<ClassLoader> classLoaders = new ArrayList<ClassLoader>();
        final String className = "ClassLoaderOOMExample";
        final byte[] code = geneDynamicClassBytes(className);

        
        while (true) {
            ClassLoaderOOMOps loader = new ClassLoaderOOMOps();
            Class<?> exampleClass = loader.defineClass(className, code, 0, code.length); 
            classLoaders.add(loader);
            
        }
    }

    private static byte[] geneDynamicClassBytes(String className) throws Exception {
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_1, ACC_PUBLIC, className, null, "java/lang/Object", null);

        
        MethodVisitor mw = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);

        
        mw.visitVarInsn(ALOAD, 0);
        mw.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(1, 1);
        mw.visitEnd();

        
        mw = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "main", "([Ljava/lang/String;)V", null, null);
        
        mw.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");

        mw.visitLdcInsn("Hello world!");
        mw.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(2, 2);
        mw.visitEnd();  

        return cw.toByteArray();  

    }
}
```

```
package com.q.mat;

import java.util.*;
import org.objectweb.asm.*;

public class ThreadAndListHolder extends ClassLoader implements Opcodes {
    private static Thread innerThread1;
    private static Thread innerThread2;
    private static final SameContentWrapperContainerProxy sameContentWrapperContainerProxy = new SameContentWrapperContainerProxy();

    static {
        // 启用两个线程作为 GC Roots
        innerThread1 = new Thread(new Runnable() {
            public void run() {
                SameContentWrapperContainerProxy proxy = sameContentWrapperContainerProxy;
                try {
                    Thread.sleep(60 * 60 * 1000);
                } catch (Exception e) {
                    System.exit(1);
                }
            }
        });
        innerThread1.setName("ThreadAndListHolder-thread-1");
        innerThread1.start();

        innerThread2 = new Thread(new Runnable() {
            public void run() {
                SameContentWrapperContainerProxy proxy = proxy = sameContentWrapperContainerProxy;
                try {
                    Thread.sleep(60 * 60 * 1000);
                } catch (Exception e) {
                    System.exit(1);
                }
            }
        });
        innerThread2.setName("ThreadAndListHolder-thread-2");
        innerThread2.start();
    }
}

class IntArrayListWrapper {
    private ArrayList<Integer> list;
    private String name;

    public IntArrayListWrapper(ArrayList<Integer> list, String name) {
        this.list = list;
        this.name = name;
    }
}

class SameContentWrapperContainer {
    // 2个Wrapper内部指向同一个 ArrayList，方便学习 Dominator tree
    IntArrayListWrapper intArrayListWrapper1;
    IntArrayListWrapper intArrayListWrapper2;

    public void init() {
        // 线程直接支配 arrayList，两个 IntArrayListWrapper 均不支配 arrayList，只能线程运行完回收
        ArrayList<Integer> arrayList = generateSeqIntList(10 * 1000 * 1000, 0);
        intArrayListWrapper1 = new IntArrayListWrapper(arrayList, "IntArrayListWrapper-1");
        intArrayListWrapper2 = new IntArrayListWrapper(arrayList, "IntArrayListWrapper-2");
    }

    private static ArrayList<Integer> generateSeqIntList(int size, int startValue) {
        ArrayList<Integer> list = new ArrayList<Integer>(size);
        for (int i = startValue; i < startValue + size; i++) {
            list.add(i);
        }
        return list;
    }
}

class SameContentWrapperContainerProxy {
    SameContentWrapperContainer sameContentWrapperContainer;

    public SameContentWrapperContainerProxy() {
        SameContentWrapperContainer container = new SameContentWrapperContainer();
        container.init();
        sameContentWrapperContainer = container;
    }
}
复制代码
```

```
启动参数：-Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/gjd/Desktop/dump/heapdump.hprof 
-XX:-UseCompressedClassPointers -XX:-UseCompressedOops
复制代码
```

**引用关系图** ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c3d2e9807374c608857ae03ddb42709~tplv-k3u1fbpfcp-watermark.image)

**分析过程**

1.  首先进入 Dominator tree，可以看出是 SameContentWrapperContainerProxy 对象与 main 线程两者持有99%内存不能释放导致 OOM。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fe24e47a05042b7b61a103e3c7dbf4b~tplv-k3u1fbpfcp-watermark.image)
     ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d953a502fdf40c6ad95a5a992ec33fb~tplv-k3u1fbpfcp-watermark.image)
    
2.  先来看方向一，在 Heap Dump Overview 中可以快速定位到 Number of class loaders 数达50万以上，这种基本属于异常情况，如下图所示。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc64b31b8e484ca88e02590b42f82c74~tplv-k3u1fbpfcp-watermark.image)
    
3.  使用 Class Loader Explorer 分析工具，此时会展现类加载详情，可以看到有524061个 class loader。我们的案例中仅有ClassLoaderOOMOps 这样的自定义类加载器，所以很快可以定位到问题。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf60312fd4894df98ccd7573a1c0c5c6~tplv-k3u1fbpfcp-watermark.image)
     ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9309e3ecfa4e48dc8b98ab2f1e4e08d6~tplv-k3u1fbpfcp-watermark.image)
    
4.  如果类加载器较多，不能确定是哪个引发问题，则可以将所有的 class loader对象按类做聚类，如下图所示。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3dad1e14bad84ced8f906bf162dad4c5~tplv-k3u1fbpfcp-watermark.image)
    
5.  Histogram 会根据 class 聚合，并展现对象数量级其 Shallow Heap 及 Retained Heap（如Retained Heap项目为空，可以点击下图中计算机的图标并计算 Retained Heap），可以看到 ClassLoaderOOMOps 有524044个对象，其 Retain Heap 占据了370M以上（上述代码是100M左右）。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09f7a1ecbe1d495cb144c16f676f9888~tplv-k3u1fbpfcp-watermark.image)
    
6.  使用 incoming references，可以找到创建的代码位置。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8128a34a4fb42a7a28d4f7c69313e12~tplv-k3u1fbpfcp-watermark.image)
    
7.  再来看方向二，同样在占据319M内存的 Obejct 数组采用 incoming references 查看引用路径，也很容易定位到具体代码位置。并且从下图中我们看出，Dominator tree 的起点并不一定是 GC根，且通过 Dominator tree 可能无法获取到最开始的创建路径，但 incoming references 是可以的。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64fac354fd204b3f93c43d48a25c0646~tplv-k3u1fbpfcp-watermark.image)
    

3.1 References
--------------

_**注：笔者使用频率 Top2**_

**功能**：在对象引用图中查看某个特定对象的所有引用关系（提供对象对其他对象或基本类型的引用关系，以及被外部其他对象的引用关系）。通过任一对象的直接引用及间接引用详情（主要是属性值及内存占用），提供完善的依赖链路详情。

**使用入口**：目标域右键 → List objects → with outgoing references/with incoming references.

**使用场景**

*   outgoing reference：查看对象所引用的对象，并支持链式传递操作。如查看一个大对象持有哪些内容，当一个复杂对象的 Retained Heap 较大时，通过 outgoing reference 可以查看由哪个属性引发。下图中 A 支配 F，且 F 占据大量内存，但优化时 F 的直接支配对象 A 无法修改。可通过 outgoing reference 看关系链上 D、B、E、C，并结合业务逻辑优化中间环节，这依托 dominator tree 是做不到的。
*   incoming reference：查看对象被哪些对象引用，并支持链式传递操作。如查看一个大对象都被哪些对象引用，下图中 K 占内存大，所以 J 的 Retained Heap 较大，目标是从 GC Roots 摘除 J 引用，但在 Dominator tree 上 J 是树根，无法获取其被引用路径，可通过 incoming reference 查看关系链上的 H、X、Y ，并结合业务逻辑将 J 从 GC Root 链摘除。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bde68d299620415cb34f5d6cbbeeeae7~tplv-k3u1fbpfcp-watermark.image)

3.2 Thread overview
-------------------

**功能**：展现转储 dump 文件时线程执行栈、线程栈引用的对象等详细状态，也提供各线程的 Retained Heap 等关联内存信息。

**使用入口**：MAT 主页 → Thread overview

**使用场景**

*   查看不同线程持有的内存占比，定位高内存消耗线程（开发技巧：不要直接使用 Thread 或 Executor 默认线程名避免全部混合在一起，使用线程尽量自命名方便识别，如下图中 ThreadAndListHolder-thread 是自定义线程名，可以很容易定位到具体代码）
*   查看线程的执行栈及变量，结合业务代码了解线程阻塞在什么地方，以及无法继续运行释放内存，如下图中 ThreadAndListHolder-thread 阻塞在 sleep 方法。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cfcded48fb234e0682bdc6d872f03717~tplv-k3u1fbpfcp-watermark.image)
    

3.3 Path To GC Roots
--------------------

**功能**：提供任一对象到 GC Root 的路径详情。

**使用入口**：目标域右键 → Path To GC Roots

**使用场景**：有时你确信已经处理了大的对象集合但依然无法回收，该功能能快速定位异常对象不能被 GC 回收的原因，直击异常对象到 GC Root 的引用路径。比 incoming reference 的优势是屏蔽掉很多不需关注的引用关系，比 Dominator tree 的优势是可以得到更全面的信息。

_小技巧：在排查内存泄漏时，建议选择 exclude all phantom/weak/soft etc.references 排除虚引用/弱引用/软引用等的引用链，因为被虚引用/弱引用/软引用的对象可以直接被 GC 给回收，聚焦在对象否还存在 Strong 引用链即可。_

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3919a0973e6f4ac0b4fe3460e24fec75~tplv-k3u1fbpfcp-watermark.image)

3.4 class loader 分析
-------------------

**功能**

*   查看堆中所有 class loader 的使用情况（入口：MAT 主页菜单蓝色桶图标 → Java Basics → Class Loader Explorer）。
*   查看堆中被不同class loader 重复加载的类（入口：MAT 主页菜单蓝色桶图标 → Java Basics → Duplicated Classes）。

**使用场景**

*   当从 Heap dump overview 了解到系统中 class loader 过多，导致占用内存异常时进入更细致的分析定位根因时使用。
*   解决 NoClassDefFoundError 问题或检测 jar 包是否被重复加载

具体使用方法在 2.6 及 3.5 两节的案例中有介绍。

3.5 综合案例二
---------

**使用工具项**：class loader（重复类检测）、inspector、正则检索。

**异常现象** ：运行时报 NoClassDefFoundError，在 classpath 中有两个不同版本的同名类。

**分析过程**

1.  进入 MAT 已加载的重复类检测功能，方式如下图。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60fbe324fe9d416c90c2e659b4f971c7~tplv-k3u1fbpfcp-watermark.image)
    
2.  可以看到所有重复的类，以及相关的类加载器，如下图。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cdae3fc7b614c01a8047e3c024b4680~tplv-k3u1fbpfcp-watermark.image)
    
3.  根据类名，在`<Regex>`框中输入类名可以过滤无效信息。
    
4.  选中目标类，通过Inspector视图，可以看到被加载的类具体是在哪个jar包里。（本例中重复的类是被 URLClassloader 加载的，右键点击 “\_context” 属性，最后点击 “Go Into”，在弹出的窗口中的属性 “\_war” 值是被加载类的具体包位置）
    
    ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db89543ca6b544cf8e36252aa08a60be~tplv-k3u1fbpfcp-watermark.image)
     ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3be497af48b4c5b9ea96b2ae2a9c967~tplv-k3u1fbpfcp-watermark.image)
    

4.1 inspector
-------------

**功能**：MAT 通过 inspector 面板展现对象的详情信息，如静态属性值及实例属性值、内存地址、类继承关系、package、class loader、GC Roots 等详情数据。

**使用场景**

*   当内存使用量与业务逻辑有较强关联的场景，通过 inspector 可以通过查看对象具体属性值。比如：社交场景中某个用户对象的好友列表异常，其 List 长度达到几亿，通过 inspector 面板获取到异常用户 ID，进而从业务视角继续排查属于哪个用户，本例可能有系统账号，与所有用户是好友。
*   集合等类型的使用会较多，如查看 ArrayList 的 size 属性也就了解其大小。

**举例**：下图中左边的 Inspector 窗口展现了地址 0x125754cf8 的 ArrayList 实例详情，包括 modCount 等并不会在 outgoing references 展现的基本属性。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5067f8c3bc94397ad829ae9ee882b34~tplv-k3u1fbpfcp-watermark.image)

4.2 集合状态
--------

**功能**：帮助更直观的了解系统的内存使用情况，查找浪费的内存空间。

**使用入口**：MAT 主页 → Java Collections → 填充率/Hash冲突等功能。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/409f4fd84bef43dfabdd9f6337615917~tplv-k3u1fbpfcp-watermark.image)

**使用场景**

*   通过对 ArrayList 或数组等集合类对象按填充率聚类，定位稀疏或空集合类对象造成的内存浪费。
*   通过 HashMap 冲突率判定 hash 策略是否合理。

具体使用方法在 4.3 节案例详细介绍。

4.3 综合案例三
---------

**使用工具项**：Dominator tree、Histogram、集合 ratio。

**异常现象** ：程序 OOM，且 Dominator tree 无大对象，通过 Histogram 了解到多个 ArrayList 占据大量内存，期望通过减少 ArrayList 优化程序。

**程序代码**

```java
package com.q.mat;

import java.util.ArrayList;
import java.util.List;

public class ListRatioDemo {

    public static void main(String[] args) {
        for(int i=0;i<10000;i++){
            Thread thread = new Thread(new Runnable() {
                public void run() {
                    HolderContainer holderContainer1 = new HolderContainer();
                    try {
                        Thread.sleep(1000 * 1000 * 60);
                    } catch (Exception e) {
                        System.exit(1);
                    }
                }
            });
            thread.setName("inner-thread-" + i);
            thread.start();
        }

    }
}

class HolderContainer {
    ListHolder listHolder1 = new ListHolder().init();
    ListHolder listHolder2 = new ListHolder().init();
}

class ListHolder {
    static final int LIST_SIZE = 100 * 1000;
    List<String> list1 = new ArrayList(LIST_SIZE); 
    List<String> list2 = new ArrayList(LIST_SIZE); 
    List<String> list3 = new ArrayList(LIST_SIZE); 
    List<String> list4 = new ArrayList(LIST_SIZE); 

    public ListHolder init() {
        for (int i = 0; i < LIST_SIZE; i++) {
            if (i < 0.05 * LIST_SIZE) {
                list1.add("" + i);
                list2.add("" + i);
            }
            if (i < 0.15 * LIST_SIZE) {
                list3.add("" + i);
            }
            if (i < 0.3 * LIST_SIZE) {
                list4.add("" + i);
            }
        }
        return this;
    }
}
```

**分析过程**

1.  使用 Dominator tree 查看并无高占比起点。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2f44f9216f94b08bc3bdf7107d5a737~tplv-k3u1fbpfcp-watermark.image)
    
2.  使用 Histogram 定位到 ListHolder 及 ArrayList 占比过高，经过业务分析很多 List 填充率很低浪费内存。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de3c98f88cf14561ae1117e2f8e55cd2~tplv-k3u1fbpfcp-watermark.image)
    
3.  查看 ArrayList 的填充率，MAT 首页 → Java Collections → Collection Fill Ratio。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44000e6e5c2946e18ed1703d68aec244~tplv-k3u1fbpfcp-watermark.image)
    
4.  查看类型填写 java.util.ArrayList。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad7db951d8254ebaa9c85f07e799f887~tplv-k3u1fbpfcp-watermark.image)
    
5.  从结果可以看出绝大部分 ArrayList 初始申请长度过大。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe6c558fd665443c874d41c920a64018~tplv-k3u1fbpfcp-watermark.image)
    

5.1 OQL
-------

**功能**：提供一种类似于SQL的对象（类）级别统一结构化查询语言，根据条件对堆中对象进行筛选。

**语法**

```sql
SELECT * FROM [ INSTANCEOF ] <class_name> [ WHERE <filter-expression> ]
```

*   Select 子句可以使用“*”，查看结果对象的引用实例（相当于 outgoing references）；可以指定具体的内容，如 Select OBJECTS v.elementData from xx 是返回的结果是完整的对象，而不是简单的对象描述信息)；可以使用 Distinct 关键词去重。
*   From 指定查询范围，一般指定类名、正则表达式、对象地址。
*   Where 用来指定筛选条件。
*   全部语法详见：[OQL 语法](https://link.juejin.cn/?target=https%3A%2F%2Fhelp.eclipse.org%2Fneon%2Findex.jsp%3Ftopic%3D%252Forg.eclipse.mat.ui.help%252Freference%252Foqlsyntax.html "https://help.eclipse.org/neon/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Freference%2Foqlsyntax.html")
*   未支持的核心功能：group by value，如果有需求可以先导出结果到 csv 中，再使用 awk 等脚本工具分析即可。
*   **例子**：查找 size＝0 且未使用过的 ArrayList：select * from java.util.ArrayList where size=0 and modCount=0。

**使用场景**

*   一般比较复杂的问题会使用 OQL，而且这类问题往往与业务逻辑有较大关系。比如大量的小对象整体占用内存高，但预期小对象应该不会过多（比如达到百万个），一个一个看又不现实，可以采用 OQL 查询导出数据排查。
*   **例子**：微服务的分布式链路追踪系统，采集各服务所有接口名，共计200个服务却采集到了200万个接口名（一个服务不会有1万个接口），这时直接在 List 中一个个查看很难定位，可以直接用 OQL 导出，定位哪个服务接口名收集异常（如把 URL 中 ID 也统计到接口中了）

5.2 检索及筛选
---------

**功能**：本文第二章内存分布，第三章对象间依赖的众多功能，均支持按字符串检索、按正则检索等操作。

**使用场景**：在使用 Histogram、Thread overview 等功能时，可以进一步添加字符串匹配、正则匹配条件过滤缩小排查范围。

5.3 按地址寻址
---------

**功能**：根据对象的虚拟内存十六进制地址查找对象。

**使用场景**：仅知道地址并希望快速查看对象做后续分析时使用，其余可以直接使用 outgoing reference 了解对象信息。

5.4 综合案例四
---------

**使用工具项**：OQL、Histogram、incoming references

**异常现象及目的** ：程序占用内存高，存在默认初始化较长的 ArrayList，需分析 ArrayList 被使用的占比，通过数据支撑是否采用懒加载模式，并分析具体哪块代码创建了空 ArrayList。

**程序代码**

```java
public class EmptyListDemo {
    public static void main(String[] args) {
        EmptyValueContainerList emptyValueContainerList = new EmptyValueContainerList();
        FilledValueContainerList filledValueContainerList = new FilledValueContainerList();
        System.out.println("start sleep...");
        try {
            Thread.sleep(50 * 1000 * 1000);
        } catch (Exception e) {
            System.exit(1);
        }
    }
}

class EmptyValueContainer {
    List<Integer> value1 = new ArrayList(10);
    List<Integer> value2 = new ArrayList(10);
    List<Integer> value3 = new ArrayList(10);
}

class EmptyValueContainerList {
    List<EmptyValueContainer> list = new ArrayList(500 * 1000);

    public EmptyValueContainerList() {
        for (int i = 0; i < 500 * 1000; i++) {
            list.add(new EmptyValueContainer());
        }
    }
}

class FilledValueContainer {
    List<Integer> value1 = new ArrayList(10);
    List<Integer> value2 = new ArrayList(10);
    List<Integer> value3 = new ArrayList(10);

    public FilledValueContainer init() {
        value1.addAll(Arrays.asList(1, 3, 5, 7, 9));
        value2.addAll(Arrays.asList(2, 4, 6, 8, 10));
        value1.addAll(Arrays.asList(1, 1, 1, 1, 1, 1, 1, 1, 1, 1));
        return this;
    }
}

class FilledValueContainerList {
    List<FilledValueContainer> list = new ArrayList(500);

    public FilledValueContainerList() {
        for (int i = 0; i < 500; i++) {
            list.add(new FilledValueContainer().init());
        }
    }
}
```

**分析过程**

1.  内存中有50万个 capacity = 10 的空 ArrayList 实例。我们分析下这些对象的占用内存总大小及对象创建位置，以便分析延迟初始化（即直到使用这些对象的时候才将之实例化，否则一直为null）是否有必要。
    
2.  使用 OQL 查询出初始化后未被使用的 ArrayList（size=0 且 modCount=0），语句如下图。可以看出公有 150 万个空 ArrayList，这些对象属于浪费内存。我们接下来计算下总计占用多少内存，并根据结果看是否需要优化。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef6b45a0681c4976a0fd59ba31de6243~tplv-k3u1fbpfcp-watermark.image)
    
3.  计算 150万 ArrayList占内存总量，直接点击右上方带黄色箭头的 Histogram 图标，这个图标是在选定的结果再用直方图展示，总计支配了120M 左右内存（所以这里点击结果，不包含 modCount 或 size 大于0的 ArrayList 对象）。**这类在选定结果继续分析很多功能都支持，如正则检索、Histogram、Dominator tree等等。**  ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99960e293cf742b3b60f90095d2c56f8~tplv-k3u1fbpfcp-watermark.image)
    
4.  查看下空 ArrayList 的具体来源，可用 incoming references，下图中显示了清晰的对象创建路径。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffe93f29a7cc499795279070ed3edb25~tplv-k3u1fbpfcp-watermark.image)
    

至此本文讲解了 MAT 各项工具的功能、使用方法、适用场景，也穿插了4个实战案例，熟练掌握对分析 JVM 内存问题大有裨益，尤其是各种功能的组合使用。在下一篇《JVM 内存分析工具 MAT 的深度讲解与实践——高阶篇》会总结 JVM 堆内存分析的系统性方法，并在更复杂的案例中实践。

*   [MAT官网](https://link.juejin.cn/?target=https%3A%2F%2Fhelp.eclipse.org%2F2020-09%2Findex.jsp%3Ftopic%3D%2Forg.eclipse.mat.ui.help%2Fwelcome.html "https://help.eclipse.org/2020-09/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html")
*   [10 Tips for using the Eclipse Memory Analyzer](https://link.juejin.cn/?target=https%3A%2F%2Feclipsesource.com%2Fblogs%2F2013%2F01%2F21%2F10-tips-for-using-the-eclipse-memory-analyzer%2F "https://eclipsesource.com/blogs/2013/01/21/10-tips-for-using-the-eclipse-memory-analyzer/")
*   [Finding Memory Leaks with SAP Memory Analyzer](https://link.juejin.cn/?target=https%3A%2F%2Fblogs.sap.com%2F2007%2F07%2F02%2Ffinding-memory-leaks-with-sap-memory-analyzer%2F "https://blogs.sap.com/2007/07/02/finding-memory-leaks-with-sap-memory-analyzer/")
*   [An effective way to fight duplicated libs and version conflicting classes using Memory Analyzer Tool](https://link.juejin.cn/?target=https%3A%2F%2Fcommunity.bonitasoft.com%2Fblog%2Feffective-way-fight-duplicated-libs-and-version-conflicting-classes-using-memory-analyzer-tool "https://community.bonitasoft.com/blog/effective-way-fight-duplicated-libs-and-version-conflicting-classes-using-memory-analyzer-tool")
*   [Memory for nothing（Empty collection problem）](https://link.juejin.cn/?target=https%3A%2F%2Fblogs.sap.com%2F2007%2F08%2F02%2Fmemory-for-nothing%2F "https://blogs.sap.com/2007/08/02/memory-for-nothing/")
