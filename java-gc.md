java 有三大知名的 jvm gc 机制:
1. SUN 的 HotSpot
2. BEA 的 JRockit
3. IBM 的 J9

自从 Oracle 把 BEA 和 sun 收了之后后, JRockit 会被融入到 HotSpot 中来, 而 J9 又因为版权只能在 IBM 授权的机器上用, 所以后面的主流还会一直是 HotSpot.

要说 jvm 就必须先讲 jmm, 其实主要就堆和栈, 本来还有寄存器区, 本地方法区(jni 调用), 

Java 在分配内存时会涉及到以下区域: http://www.cnblogs.com/paddix/p/5309550.html
1. PC 寄存器(Program Counter Register)
```
程序计数器, 普通方法则保存当前执行指令的地址(字节码行号指示器), native 方法则为空
```
2. 虚拟机栈(VM Stacks):
```
方法执行时的内存模型: 局部变量(基本类型和对象引用), 操作数, 动态链接, 方法出口
```
3. 本地方法栈(Native Method Stack):
```
本地 native 方法
```
4. 堆(Heap):
```
虚拟机管理的最大一块内存. 存放对象. gc 的重点就是操作这部分

堆(Heap)又分为(两/三)个区域:
  新生代(Young) 它又划分为 Eden(伊甸)、From Survivor(存活者)和 To Survivor(遗孀)三个区域
  老年代(Old Generation) 新生代进入老年代的阀值设置 -XX:MaxTenuringThreshold= 默认是 15

From Survivor 和 To Survivor 只是算法在收集时的主从关系而已, 两者的界定并没有很明确.

在年轻代(Young)中发生的 GC 又称为小回收(Minor GC), 这部分是最为频繁的, 通常是采用复制收集算法.
在老年代(Old)中发生的 GC 称为大回收(Major GC).
完全回收(Full GC)时会同时进行至少一次 Minor GC, 通常在堆内存紧张或显示调用 System.gc() 时触发(-XX:+DisableExplicitGC 将会忽略这种显式调用 gc 的情况)

这种针对区域进行不同回收的技术又叫分代回收(Generational GC)
```
5. 方法区(Method Area):
```
在 HotSpot 中又叫永久代(Permanent Generation), JRockit 和 J9 并没有 PermGen space, 
主要存储已经被虚拟机加载的类信息, 常量(字符串常量池), 静态变量, 即时编译后的代码等, 
对这一区域的回收主要是针对常量和对类卸载, Java8 开始改成了 Metaspace(元空间)

在 jdk7 中永久代是存在于 Heap 中的, Java8 开始改成了 Metaspace(元空间)也是在堆中,
像       -XX:PermSize=1m      -XX:MaxPermSize=1m 这些永久代的配置也不再生效,
替换成了 -XX:MetaspaceSize=1m -XX:MaxMetaspaceSize=1m
```

PS: 前面两个区域不受应用程序管控, 前三个区域是线程私有的(运行线程在就有, 线程执行完就能被回收), 后面两个是线程共享

另外: 永久代的垃圾回收和老年代的垃圾回收是绑定的, 一旦其中一个区域被占满, 这两个区都要进行垃圾回收.

但是有一个明显的问题, 由于我们可以通过‑XX:MaxPermSize 设置永久代的大小,
一旦类的元数据超过了设定的大小, 程序就会耗尽内存, 并出现内存溢出错误(OOM).


堆是公共区域, 栈是线程私有区域. 比如一个方法里面的临时变量 `String a = "123";` a 这个引用(也就是 等号 左边的值)就是放在栈里面的,
当方法所在的线程执行完就会被抹掉. 这一块不够用的情况并不多见, 当方法递归(方法调用的行为也是在栈里面)层次太多会抛出 `StackOverflowError`

堆的结构从 dump log 里面可以有一个大致印象
```
Heap
  PSYoungGen
    eden space
    from space
    to   space
  ParOldGen
    Object space
  Metaspace
    Class  space
```

1.8 之前 metaspace 又叫 永久代(Permanent Generation, JRockit 和 J9 里面管这块叫方法区 Method Area). 主要存的是 java 的第一公民「类」相关的信息, 这块区域是很难做 gc 的, 现在类的动态加载越来越多(第三方语言 scala kotlin groovy 以及 asm 字节码增强等), 1.8 开始的 gc 已经开始加强了这一块的管理.

只针对 young 的叫 minor gc, 针对 old 的叫 major gc, 因为 major 的时候一定会先来一次 minor, 所以又叫 full gc, 这样分块 gc 的概念又叫分代回收.

通常 System.gc() 都是会触发 gc 的, 除非启动 java 命令的时候加一个抑制的命令(-XX:+DisableExplicitGC), 一块空间经过了多次 minor 还活着就转到 major 的阀值默认是 15, 选项: -XX:MaxTenuringThreshold=20

判断一块空间是不是 gc 目前还只有两种方法, 一个是可达性分析, 一个是计数器.


基于确定是不是垃圾的这两种方法(可达性分析和计数器), 一共衍生出三种 Garbage Collection 算法(松本行弘 --> 代码的未来):

1. 标记清除(Mark and Sweep). 两次循环, 第一次从根对象开始遍历并标记, 第二次遍历整个堆, 没有标记(表示无法到达)的就回收
```
标记清除算法有一个缺点, 当分配了大量对象, 并且其中只有一小部分存活的情况下,
所消耗的时间会很没有必要, 因为清除阶段需要对大量死亡对象进行扫描.
而且标记清除容易出现成内存碎片, 作为标记清除的变形,
还有一种叫做标记压缩(Mark And Compact)的算法, 它不是将被标记的对象清除, 而是将它们不断压缩.
```
2. 复制收集(Copy and Collection). 这是一种很大胆的算法, 从根开始把整颗树复制到一个新地方, 再把原来那份的空间整个回收
```
复制收集算法在存在大量对象, 且其中大部即将死亡的情况下, 全部扫描一遍的开销会少很多, 
当然, 如果存活对象比例较高时, 反而会比较不利,
它的另一个好处是局部性, 关系较近的对象被放在距离较近的内存空间的可能性会提高, 这被称为局部性. 
局部性高的情况下, 内存缓存会更容易有效运行, 程序的运行性能也能够得到提高.
```
3. 引用记数(Reference Count). 对象每调用一次就递增一次引用数, 回收数值为 0 的对象即可.
```
实现容易是引用计数算法最大的优点, 这种算法相当具有普通性
除此之外, 当对象不再被引用的瞬间就会被释放, 这也是一个优点.
由于释放操作是针对每个对象个别执行的, 因此由 GC 而产生的中断时间(Pause time)就比较短, 这同样是一个优点.
它最大的缺点就是无法释放循环引用的对象, 第二个缺点是必须在引用发生增减时就做出正确的增减, 如果漏掉就会引发很难找到原因的内存错误.
```

其他的 gc 算法, 大体上都逃不出上述三种方式以及它们的衍生品.


是不是垃圾和上面的三种算法, 都差不多是同一时期发现的, 直到现在都还没有人发现一个新的方法. 如果有, 发个 pager, 没准能得图灵奖.

jvm 里面, 新生代的算法是基于 复制-收集 来实现, 从 from to 这两区域的名称基本就能感觉得出来. 老年代是基于 标记-压缩 来实现.

-----

Java 有四种类型的垃圾回收器:
1. 串行垃圾回收器(Serial Garbage Collector) 单线程且会冻结所有正在运行的线程, 会 stw(stop the world)
2. 并行垃圾回收器(Parallel) 默认的 gc 回收器, 使用多线程, 会 stw
3. 并发标记扫描垃圾回收器(CMS). 并发标记, 多线程扫描. 清理标记过的实例, 能根据情况在回收时只冻结相关的线程
4. G1 垃圾回收器(G1) jdk 7 出现, 用来取代 cms, 并行和并发, 分代处理, 内存整理, 更多的预见性来消除内存碎片

运行的垃圾回收器类型

| 配置 | 描述 |
| ---- | ---- |
| -XX:+UseSerialGC | 串行垃圾回收器 |
| -XX:+UseParallelGC | 并行垃圾回收器 |
| -XX:+UseConcMarkSweepGC | 并发标记扫描垃圾回收器 |
| -XX:ParallelCMSThreads=xxx | 并发标记扫描垃圾回收器 =为使用的线程数量 |
| -XX:+UseG1GC | G1垃圾回收器 |

-----

eclipse config(来自《深入理解 jvm》第 160 页):
```conf
# 初始堆内存大小
-Xms512m
# 堆内存最大值
-Xmx215m
# 新生代大小
-Xmn256m
# 不让虚拟机进行字节码验证, 一般都由 eclipse 编译或 maven
-Xverify:none
# 永久代大小
-XX:PermSize=128m
# 永久代最大值
-XX:MaxPermSize=128m
# 禁用 System.gc()
-XX:+DisableExplicitGC
# 忽略类信息的垃圾回收功能, 主要指一些动态代理生成的类
-Xnoclassgc
# 设置年轻代为并行收集, 可与 cms 同时使用
-XX:+UseParNewGC
# 使用标记扫描垃圾回收器
-XX:+UseConcMarkSweepGC
# 堆空间使用 85% 后开始 cms 收集
-XX:CMSInitiatingOccupancyFraction=85
```
http://icyfenix.iteye.com/blog/1256329

-----

各种 gc 测试

http://www.optaplanner.org/blog/2015/07/31/WhatIsTheFastestGarbageCollectorInJava8.html

http://blog.takipi.com/garbage-collectors-serial-vs-parallel-vs-cms-vs-the-g1-and-whats-new-in-java-8/

https://plumbr.eu/blog/garbage-collection/g1-vs-cms-vs-parallel-gc

-----

| 配置 | 描述 |
|-----|-----|
| -Xms | 初始化堆内存大小 |
| -Xmx | 堆内存最大值 |
| -Xmn | 新生代大小 |
| -XX:PermSize | 初始化永久代大小 |
| -XX:MaxPermSize | 永久代最大容量 |
| -XX:G1HeapRegionSize=n | 设置的 G1 区域的大小. 值是 2 的幂, 范围是 1 MB 到 32 MB 之间. 目标是根据最小的 Java 堆大小划分出约 2048 个区域 |
| -XX:MaxGCPauseMillis=200 | 为所需的最长暂停时间设置目标值. 默认值是 200 毫秒. 指定的值不适用于您的堆大小 |
| -XX:G1NewSizePercent=5 | 设置要用作年轻代大小最小值的堆百分比. 默认值是 Java 堆的 5%. 这是一个实验性的标志. 有关示例, 请参见“如何解锁实验性虚拟机标志”. 此设置取代了 -XX:DefaultMinNewGenPercent 设置. Java HotSpot VM build 23 中没有此设置 |
| -XX:G1MaxNewSizePercent=60 | 设置要用作年轻代大小最大值的堆大小百分比. 默认值是 Java 堆的 60%. 这是一个实验性的标志. 有关示例, 请参见“如何解锁实验性虚拟机标志”. 此设置取代了 -XX:DefaultMaxNewGenPercent 设置. Java HotSpot VM build 23 中没有此设置 |
| -XX:ParallelGCThreads=n | 设置 STW 工作线程数的值. 将 n 的值设置为逻辑处理器的数量. n 的值与逻辑处理器的数量相同, 最多为 8. 如果逻辑处理器不止八个, 则将 n 的值设置为逻辑处理器数的 5/8 左右. 这适用于大多数情况, 除非是较大的 SPARC 系统, 其中 n 的值可以是逻辑处理器数的 5/16 左右 |
| -XX:ConcGCThreads=n | 设置并行标记的线程数. 将 n 设置为并行垃圾回收线程数 (ParallelGCThreads) 的 1/4 左右 |
| -XX:InitiatingHeapOccupancyPercent=45 | 设置触发标记周期的 Java 堆占用率阈值. 默认占用率是整个 Java 堆的 45% |
| -XX:G1MixedGCLiveThresholdPercent=65 | 为混合垃圾回收周期中要包括的旧区域设置占用率阈值. 默认占用率为 65%. 这是一个实验性的标志. 有关示例, 请参见“如何解锁实验性虚拟机标志”. 此设置取代了 -XX:G1OldCSetRegionLiveThresholdPercent 设置. Java HotSpot VM build 23 中没有此设置 |
| -XX:G1HeapWastePercent=10 | 设置您愿意浪费的堆百分比. 如果可回收百分比小于堆废物百分比, Java HotSpot VM 不会启动混合垃圾回收周期. 默认值是 10%. Java HotSpot VM build 23 中没有此设置 |
| -XX:G1MixedGCCountTarget=8 | 设置标记周期完成后, 对存活数据上限为 G1MixedGCLIveThresholdPercent 的旧区域执行混合垃圾回收的目标次数. 默认值是 8 次混合垃圾回收. 混合回收的目标是要控制在此目标次数以内. Java HotSpot VM build 23 中没有此设置 |
| -XX:G1OldCSetRegionThresholdPercent=10 | 设置混合垃圾回收期间要回收的最大旧区域数. 默认值是 Java 堆的 10%. Java HotSpot VM build 23 中没有此设置 |
| -XX:G1ReservePercent=10 | 设置作为空闲空间的预留内存百分比, 以降低目标空间溢出的风险. 默认值是 10%. 增加或减少百分比时, 请确保对总的 Java 堆调整相同的量. Java HotSpot VM build 23 中没有此设置 |

