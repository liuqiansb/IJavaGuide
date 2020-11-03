##### 关于JVMGC的问题

- CMS和G1的异同

- G1什么时候触发full gc

- 说一个最熟悉的垃圾回收算法

- 吞吐量优先和响应时间优先的回收器有哪些

- 怎么判断内存泄漏

- 讲一下CMS的流程

- 为什么压缩指针超过32G失效

- 什么是内存泄露,GC调优有经验吗

  该释放的内存没有被释放

- ThreadLocal有没有内存泄漏的问题?

- G1两个Region不是连续的，而且之间还有可达引用，我现在要回收一个，另外一个怎么处理

- 讲一下JVM的堆内存管理

- 听说过CMS的并发预处理和并发可终端预处理吗

- 到底多大的对象会被直接丢到老年代

- 如何证明你jvm调优很6

  精通十种垃圾回收器算法



##### 基本知识

- 垃圾标记的方式

  1. 引用计数

     对于一个变量，当它的引用数降为0的时候判断为垃圾，但是这个无法解决一堆垃圾循环引用的问题

  2. 根可达 Root Searching
     当一个线程栈完事的时候，我们可以把这个线程栈中所有的变量都gc掉，但在方法栈运行中，**从根上开始找，不可达的变量将会被标记为垃圾**

- 垃圾清除的方法

  1. Mark-Sweep标记清除法

     直接将可达的标记为有效，其余垃圾直接干掉，**会产生内存碎片**

  2. Copying 复制法

     复制一块同等大小的内存，将可达的复制到新内存中，然后把旧内存所有数据干掉，**比较浪费空间**

  3. Mark-Compact 标记压缩

     碎片整理，边干掉垃圾，边除掉空闲碎片，**效率低**

- 垃圾回收器

  1. Serial  a stop-the-world ,copying collector which uses a single gc thread   

  2. Serial Old  a stop-the-world mark-sweep-compact collector that uses single gc thread

     **单线程的垃圾回收方式，停止工作线程，在单cpu的情况下效率最高**，支持个几十M上百M，如果内存大了，stw时间太长

  3. Parallel Scavenge a stop-the-world copying collector which uses multiple gc threads

  4. Parallel Old a compacting collector that uses multiple gc thread

     **多线程的垃圾回收方式，停止工作线程**，但是如果内存太大了，还是会占用很长的stw时间

  5. ParNew a stop-the-world copying collector which uses multiple gc threads

     它和ps的唯一区别是它专门为了配合cms设计

  6. CMS concurrent mark sweep  a mostly concurrent **low-pause** collector  

     **不再需要太长时间的stw** **具有跨时代意义的垃圾回收方式，从它开始了并发的垃圾回收**，一个gc操作在标记的过程中寻路是最耗时的，如果我们先标记了根对象，然后再并发查找子引用对象，将节省大量时间

     1. initial mark  标记根上面的引用对象 ，主线程停止

     2. concurrent mark  标记根上对象的子引用对象，主线程并行

        问题：concurrent mark执行时，initial标记的有效对象被弃用，或无效对象被重新启用，这个咋办,这样就引入了重新标记的概念

     3. remark  重新标记根对象，并修复在concurrent mark过程中误标的对象

     4. concurrent sweep  并发清除

     ****

     **弊端：浮动垃圾，浮动垃圾产生后使用一个单线程的Serial Old来处理,碎片化**，所以CMS一卡就是卡一天，所以没有任何一个版本的java使用的是CMS，CMS需要大内存

     ![IMG20201101_032627](d:\user\01385150\desktop\IMG20201101_032627.jpg)

  ![IMG20201101_013644](d:\user\01385150\desktop\IMG20201101_013644.jpg)

  1.8默认用的ps-po,1.0自带的用的ss,后面用版本的par-cms，目前用的多的是ps-po和ss，调优的时候用的par-cms，**只要是1.8，建议使用G1**

  **记住：最好的调优是重启**

  ------

  

- 三色标记法

  cms和g1都用

- 分代回收

  1. 注意在1.8默认的ps-po垃圾回收器组合的时候，年龄为15的将扔到老年代，而cms为6

  2. 设计老年代的目的主要是，如果一个对象怎么都回收不掉，那么每次gc都去操作它是没有必要且耗费性能，所以扔到老年代，等到空间不够的时候再full gc
  3. 青年代使用copying回收算法，因为快且频繁，老年代的回收使用Mark Compact 或者 Mark Sweep

  ![IMG20201101_022124](d:\user\01385150\desktop\IMG20201101_022124.jpg)

- 对象分配原理

  **如果开启栈上分配，JVM会先进行栈上分配，以后别再说复杂对象是在堆上了，也可能在栈上**，如果没有开启栈上分配或则不符合条件的则会进行TLAB分配，

  如果TLAB分配不成功，再尝试在eden区分配，如果对象满足了直接进入老年代的条件，那就直接分配在老年代。

  **为什么：分配在栈上的好处是可以在函数调用结束后自行销毁，而不需要垃圾回收器的介入，从而提高系统性能**。 

  问题：逃逸分析,标量替换 ,如果对象在栈帧外引用？分配在栈中的对象如果有两个属性，将会以分别存两个局部变量的形式来存储这个对象

  1. minor gc  年轻代回收

  2. major gc  老年代回收

  3. full gc       老年代和年轻代一起回收

  4. TLAB  thread local alocation buffer 线程本地缓存

     背景：在new一个对象的时候，需要抢占一处空间，然而两个new不能搞到一块空间去，这就必须进行线程同步，然而new又是一个非常常见且频繁的操作，所以在高并发的情况下，同步的代价太大

     所以优化之后，**在eden区域为每一个线程先分配了一个TLAB，这样就不会去抢别的线程的空间了**，减少了同步线程的代价

     **TLAB空间的内存非常小，缺省情况下仅占有整个Eden空间的1%，可以通过参数配置**

     无论是minor GC还是full GC，要收集Eden的时候里面的空间无论是属于某个线程的TLAB还是不属于任何TLAB都一视同仁，**把Eden当作一个整体来收集里面的对象**，当gc结束的时候，每个线程又需要重新分配它的TLAB，TLAB只是在第一次new的时候用到，当gc触发的时候，原来西安城的TLAB将全部被干掉，所以它被称为缓冲区







##### 

操作：装车记录上传场院 task模块

EOS_TDOP_CORE_LOADCAR_TO_TDYM 132条报错记录

操作：装笼 装车container模块  219546条报错记录

EOS_TDOP_CORE_LOAD_CAGE_TO_TDYM 

EOS_TDOP_CORE_LOADCAR_TO_TDYM 

