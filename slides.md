---
# try also 'default' to start simple
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
css: unocss
---

# 性能检测利器和APM方案

全面了解现有性能检测手段，量化性能指标，分析一些案例，以及分析有名APM方案实现的原理

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    让我们开始吧 <carbon:arrow-right class="inline"/>
  </span>
</div>
<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/Nemo-Li/AndroidPerformance" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>
<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade-out
---

**卡顿**

应用优化最主要的方面是卡顿优化，内存和CPU都会对卡顿产生影响。本次分享从卡顿入手，引申出内存和CPU对卡顿的影响原理以及具体案例的分析

**卡顿的定义**

卡顿从视觉层面分析的直接原因为从屏幕看到的画面出现了多帧相同的情况。即通常说的掉帧，当前需要显示的帧没有准备好而显示了上一帧。

从程序运行角度，程序运行耗时，画面渲染数据在垂直同步信号来时没有准备完成。

💡 帧率越高越流畅吗？

由于人类视觉暂留对变化比较敏感，所以稳定的FPS流畅，不稳定的FPS就比较卡顿。所以并不是帧率越高越流畅，需要保证有一个稳定的帧率。Android的垂直同步信号就是为了能够按照稳定的屏幕刷新率进行界面展示。

因此，对于大多数的Android设备来说，渲染帧数据需要在16.6ms内准备完毕，这样才能保证稳定的60FPS运行。*当然如果卡顿的很稳定，画面可以稳定在一个固定的FPS，应该也是显示流畅的*

卡顿都是表现在动画过程中，静止界面看不出卡顿

<style>
  p {
    font-color:#ff0000;
  }
</style>

<!--
屏幕刷新率和vsync频率有关系吗？ 一个是屏幕硬件刷新画面的频率，一个是系统发送交换前后缓冲区的频率。理论上并没有绑定关系，只是一般情况下两者相等

-->

---
transition: slide-up
---

**卡顿的标准**

参考Google的Android Vitals性能指标和腾讯的PrefDog性能指标，给出通用应用和游戏的参考标准

**通用应用卡顿**

Google Vitals定义了两种卡顿程度

- **呈现速度缓慢**： 当超过50%的帧呈现时间超过16ms，用户感官明显卡顿
- **帧冻结**：绘制耗时超过700ms，为严重卡顿问题
- **卡顿忽略**： fps≤2的画面，人眼的视觉暂留100~400ms，对应fps为2.5~10之间。所以fps低于3，人眼看到的不是连续动作，即使有丢帧也不会察觉。

**游戏应用的卡顿**

PerfDog的Jank计算方法 (同时满足两条件) 

- **普通卡顿 Jank**  (1)当前帧耗时>前三帧平均耗时2倍 (2)当前帧耗时>两帧电影帧耗时(1000ms/24*2 = 84ms)
- **严重卡顿 BigJank**   (1)当前帧耗时>前三帧平均耗时2倍  (2)当前帧耗时>三帧电影帧耗时(125ms)

---
layout: image-right

image: /kadun.png

---
## 引起卡顿的原因
### 内存方面

STW 内存GC引起的STW现象（stop the world）

### CPU

等待CPU时间片，线程过多引起的时间片抢占

IO阻塞时间，锁的问题

所有对主线程有阻塞的操作都会导致卡顿

### 细分卡顿原因

system_server引起的应用卡顿

Input事件处理引起的卡顿

---

## Systrace

Systrace 中执行线程时间片颜色解析

<div style="width:10px;height:10px;background-color:#AADAB6;display: inline-block;"></div>
<div style="display: inline-block;padding-left:10px;font-weight:bold">绿色 Running</div>

​	线程正在完成与进程相关的工作或正在响应中断。

<div style="width:10px;height:10px;background-color:#738DC8;display: inline-block;"></div>
<div style="display: inline-block;padding-left:10px;font-weight:bold">蓝色 Runnable</div>

​	该线程可以运行，但当前未安排。（时间片被其它的线程抢占）

<div style="width:10px;height:10px;background-color:#ECECEC;display: inline-block;"></div>
<div style="display: inline-block;padding-left:10px;font-weight:bold">白色 sleeping</div>

​	线程没有工作要做，可能是因为线程被互斥锁阻塞了。

<div style="width:10px;height:10px;background-color:#FC780A;display: inline-block;"></div>
<div style="display: inline-block;padding-left:10px;font-weight:bold">橙色 Uninterruptible Sleep - Block I/O</div>

​	线程在 I/O 上阻塞或等待磁盘操作完成。

<div style="width:10px;height:10px;background-color:#A4677C;display: inline-block;"></div>
<div style="display: inline-block;padding-left:10px;font-weight:bold">紫色 Uninterruptible Sleep</div>

	线程被另一个内核操作阻塞，通常是内存管理。
---

# 案例分析

## 锁优化

以下针对launcher抓取的systrace文件进行分析

整体来看比较流畅，掉帧情况比较少，下面搜索monitor，针对锁的问题排查原因

<div grid="~ cols-3 gap-2">
	<img border="rounded" src="/launcher-trace.png">
	<img border="rounded" src="/mq-one.png">
	<img border="rounded" src="/mq-two.png">
</div>


<style>
  div {
    background:#eeeeee;
  }
</style>

---

<div>
  <img src="/launcher-analyze.png" border="rounded" style="width:50%"/>
</div>


这里launcher的主线程被background.statics.boot阻塞。UI thread需要调用MessageQueue的removeMessages方法，但是这时MessageQueue的同步锁被background.statics.boot线程的MessageQueue.enqueueMessage方法持有，导致UI线程进入锁池等待。进而影响这一帧的整体时长超过了16.6ms，产生了Jank。

通过上面现象进行问题定位分析：主线程MessageQueue的锁被子线程background.statics.boot持有，说明该子线程内部有调用了主线程的handler来处理问题导致子线程持锁。代码中对子线程内部处理代码逐个排查，发现有可疑点如上图所示

<style>
  p {
    text-indent:2em;
  }
</style>


---

### 内存相关理论和原理

<br>
<div grid="~ cols-2 gap-2">
	<div style="box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2), 0 6px 20px 0 rgba(0, 0, 0, 0.19);width:80%;margin-bottom: 25px;">
	  <img src="/jvm-memory.jpeg" border="rounded"/>
	  <div style=" text-align: center;padding: 10px 20px;">
	  <p>JVM虚拟机内存区域</p>
	  </div>
	</div>
	<div style="font-size:small">
		<p>JVM 内存区域主要分为线程私有区域【程序计数器、虚拟机栈、本地方法区】、线程共享区域【JAVA 堆、方法区】、直接内存。</p>
    <p>线程私有数据区域生命周期与线程相同, 依赖用户线程的启动/结束 而 创建/销毁(在 Hotspot VM内, 每个线程都与操作系统的本地线程直接映射, 因此这部分内存区域的存/否跟随本地线程的生/死对应)。</p>
    <p>线程共享区域随虚拟机的启动/关闭 而 创建/销毁</p>
    <p><span style="color:red">直接内存并不是 JVM 运行时数据区的一部分,</span> 但也会被频繁的使用: 在 JDK 1.4 引入的 NIO 提供了基于 Channel 与 Buffer 的 IO 方式, 它可以使用 Native 函数库直接分配堆外内存, 然后使用DirectByteBuffer 对象作为这块内存的引用进行操作。这样就避免了在 Java堆和 Native 堆中来回复制数据, 因此在一些场景中可以显著提高性能。 </p>
    <p style="border-radius:8px;padding:10px;background-color:#eaffd0;background-origin:border-box;font-size:12px">jvm 内存可视化工具jvisualvm，可以查看内存分区具体 <br>
      /Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home/bin
    </p>
	</div>
</div>

---

##### 程序计数器(线程私有)

一块较小的内存空间, 是当前线程所执行的字节码的行号指示器，每条线程都要有一个独立的 

程序计数器，这类内存也称为“线程私有”的内存。

这个内存区域是唯一一个在虚拟机中没有规定任何 OutOfMemoryError 情况的区域。

##### 虚拟机栈（线程私有）

是描述java方法执行的内存模型，每个方法在执行的同时都会创建一个栈帧（Stack Frame） 

用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成 

的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

生命周期与线程相同，每个方法执行同时创建一个栈帧。

HotSpot虚拟机中并不区分虚拟机栈和本地方法栈，两个合二为一

栈深度：栈的高度称为栈的深度，（简单理解：栈总内存/单个栈帧内存）。所以栈深度受栈帧大小影响，栈帧占用内存越大，整个栈的深度就越小

##### 本地方法区（线程私有）  

本地方法区和 Java Stack 作用类似, 区别是虚拟机栈为执行 Java 方法服务, 而本地方法栈则为 

Native 方法服务

##### 堆（Heap-线程共享）-运行时数据区

被线程共享的内存区域，现代VM采用分代收集算法，从GC角度可以细分为：新生代和老年代

方法区/永久代（线程共享）

##### 永久代（Permanent Generation）

用于存储被JVM加载的类信息、常量、静态变量、JIT编译后的代码等数据。HotSpot VM把GC分代收集扩展至方法区。

<style>
  p {
    font-size:12px;
    height:10px;
    text-indent:1em;
  }
  h5 {
    height:14px;
  }
</style>
---

### JVM运行时内存

<div grid="~ cols-2 gap-2">
	<img border="rounded" src="/jvm-one.png">
	<img border="rounded" src="/jvm-two.png">
</div>

**新生代（Young Generation）**

新生代用来存放新创建的对象，对象创建操作频繁，所以新生代会频繁触发MinorGC进行垃圾回收。新生代分为Eden、ServivorFrom、ServivorTo三个区。

Eden区：新创建对象区，如果新创建对象占用内存很大，直接分配到老年代。当Eden区内存不够就会触发MinorGC，对新生代进行一次垃圾回收

ServivorFrom：上一次GC的幸存者，作为这一次GC的被扫描者。

ServivorTo: 保留了一次MinorGC过程中的幸存者

MinorGC的过程（复制-> 清空 -> 互换）

MinorGC 采用复制算法。

<style>
  p {
    font-size:15px;
  }
</style>

---

**老年代 （Old Generation）**

主要存放应用程序中生命周期长的内存对象

老年代的对象比较稳定，所以 MajorGC 不会频繁执行。在进行 MajorGC 前一般都先进行 

了一次 MinorGC，使得有新生代的对象晋身入老年代，导致空间不够用时才触发。当无法找到足 

够大的连续空间分配给新创建的较大对象时也会提前触发一次 MajorGC 进行垃圾回收腾出空间。 

MajorGC 采用标记清除算法：首先扫描一次所有老年代，标记出存活的对象，然后回收没 

有标记的对象。MajorGC 的耗时比较长，因为要扫描再回收。MajorGC 会产生内存碎片，为了减 

少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。当老年代也满了装不下的 

时候，就会抛出 OOM（Out of Memory）异常。

**永久代 （Permanent Generation）**

指内存的永久保存区域，主要存放 Class 和 Meta（元数据）的信息,Class 在被加载的时候被 

放入永久区域，它和和存放实例的区域不同,GC 不会在主程序运行期对永久区域进行清理。所以这 

也导致了永久代的区域会随着加载的 Class 的增多而胀满，最终抛出 OOM 异常。

<style>
  p {
    font-size:15px;
    height:18px;
  }
</style>

---

### 确认垃圾对象

<br>

**引用计数法**

Java中，引用和对象是有关联的。如果要操作对象则必须用引用进行。因此，很显然一个简单 

的办法是通过引用计数来判断一个对象是否可以回收。引用计数为0则可以进行回收，在C++中，智能指针就是通过引用计数来进行内存回收的

**可达性分析**

为了解决引用计数法的循环引用问题，Java 使用了可达性分析的方法。通过一系列的“GC roots” 

对象作为起点搜索。如果在“GC roots”和一个对象之间没有可达路径，则称该对象是不可达的。

要注意的是，不可达对象不等价于可回收对象，不可达对象变为可回收对象至少要经过两次标记过程。两次标记后仍然是可回收对象，则将面临回收。

---

**GC root对象：**

1. **虚拟机栈中引用的对象**
   - 各个线程被调用的方法中使用到的参数、局部变量
2. **本地方法栈内JNI引用的对象**
3. **方法区中类静态属性引用的对象**
   - Java类中引用类型的静态成员变量
4. **方法区中常量引用的对象**
   - 字符串常量池里的引用
5. **所有被同步锁synchronized持有的对象**
6. **Java虚拟机内部的引用**
   - 基本数据类型对应的Class对象，一些常驻的异常对象（如：NullPointerException、OutOfMemoryError），系统类加载器。
7. **反映java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等**
8. **除了这些固定的GCRoots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整GC Roots集合。比如：分代收集和局部回收（Partial GC）。**

**小结：常见的root就是栈中的变量和静态变量**

<style>
  div {
    font-size:16px;
    height:18px;
  }
</style>
---
layout: two-cols
---

#### 垃圾回收算法

##### 复制算法（copying）

1. eden、servivorFrom复制到servivorTo，年龄加1

首先，把 Eden 和 ServivorFrom 区域中存活的对象复制到 ServivorTo 区域（如果有对象的年龄已经达到了老年的标准，则赋值到老年代区），同时把这些对象的年龄+1（如果 ServivorTo 不够位置了就放到老年区）；

2. 清空eden、servivorFrom

然后清空eden和servivorFrom中的对象

3. servivorFrom和servivorTo互换

最后，ServicorTo 和 ServicorFrom 互换，原 ServicorTo 成为下一次 GC 时的 ServicorFrom区

::right::

![](/copying.gif)

优点：实现简单，内存效率高，不易产生碎片

缺点：可用内存被压缩，存活对象较多的话，copying算法的效率会比较低

<style>
  p {
    margin-right:20px;
  }
</style>

---

##### 标记清除算法（Mark-Sweep）

最基础的垃圾回收算法，分为两个阶段，标注和清除。标记阶段标记出所有需要回收的对象，清除阶段回收被标记的对象所占用的空间。如图:

<style>
  p {
    text-indent:2em;
  }
</style>
<div align="center">
  <img src="/mark-sweep.gif" border="rounded" style="width:45%"/>
</div>


优点：执行效率高 

缺点：内存的碎片化比较严重

---

**标记整理算法（Mark-Compact）**

结合了以上两个算法，为了避免缺陷而提出。标记阶段和 Mark-Sweep 算法相同，标记后不是清理对象，而是将存活对象移向内存的一端。然后清除端边界外的对象。如图：

<div align="center">
  <img src="/mark-compact.gif" border="rounded" style="width:45%"/>
</div>



优点： 没有内存碎片，内存使用率高

缺点： 执行效率比Mark-Sweep低

<style>
  p {
    text-indent:2em;
  }
</style>
---
layout: cover
---

# 分代收集算法
根据不同代，采用上述不同的算法。分代收集法是目前大部分 JVM 所采用的方法，其核心思想是根据对象存活的不同生命周期将内存划分为不同的域。老生代的特点是每次垃圾回收时只有少量对象需要被回收，新生代的特点是每次垃圾回收时都有大量垃圾需要被回收，因此可以根据不同区域选择不同的算法。

新生代使用复制算法，老年代使用标记整理算法

---

## 开源APM方案预判内存泄漏

**主动检测法**

LeakCanary中的实现

- Activity 的检测预判
- Service 的检测预判
- Bitmap 大图的检测预判

1、Activity 的检测预判 LeakCanary 中对 Activity 的预判是在 onDestroy 生命周期中通过弱引用队列来持有当前 Activity 引用，如果在主动触发 gc 之后，泄漏对象集合中仍然能找到该引用实例，则说明发生了内存泄漏，就开始 dump 

2、Service 的检测预判 LeakCanary 对 Service 的内存泄漏检测时机，是 hook 监听 ActivityThread 的 stopService，然后记录这个 binder 到弱引用集合中，然后代理 AMS 的 serviceDoneExecuting 方法，通过 binder 在弱引用集合中去移除，移除成功的话，说明发生了内存泄漏，就开始 dump 

3、Bitmap 大图检测预判 Bitmap 不像 Activity、Service 这种，能够通过生命周期主动监测当前是否有内存泄漏的可能，他一般是在 Activity、Service 发生泄漏 dump 的时候，顺便检测一下 Bitmap 。在 Koom 中，Bitmap 大图检测是分析 hprof 中是否有超过 Bitmap 设置的阈值 size (width * height) 

**阈值检测法**

阈值检测法的代表框架是 Koom，他抛弃了 LeakCanary 的实时检测性，采用定时轮训检测当前内存是否在不断累加，增长达到一定次数(可自己配置)时会进行 dump hprof，这种方式会牺牲一定的时效性，但对于应用到线上的 Koom 的框架，他完全不需要这么高的时效性

<style>
  div {
    font-size: 12px
  }
</style>

---

**shallow heap和retained heap**

shallow heap 表示对象本身占用的内存大小，不包括它引用的实例。

```shell
Shallow Size = [类定义] + 父类fields所占空间 + 自身fields所占空间 + [alignment]
```

类定义固定为8byte，field分基本类型和引用类型，引用固定为4byte，基本类型和系统有关，32位系统int=4byte

retained heap 自身内存加上引用的内存

<div align="center">
  <img src="/heap.png" style="width:30%"/>
</div>

假设O1-O4的shallow heap都是1kb，以下为retained heap值

O1: 没有引用，1kb<br>O3: 引用O4， 2kb<br>O2: 引用O3，但是O1引用O3，所以O2释放O3不一定释放，1kb<br>O1: 引用O2、O3，O3引用O4，所以值为 4kb

<style>
  p {
    font-size: 16px;
    line-height: 14px;
  }
</style>

---

## JVM知识点补充

JVM有基于栈的指令集和基于寄存器的指令集<br>android平台的Dalvik VM就是基于寄存器的虚拟机

案例演示

```shell
javap -c -v Accumulate
Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: iconst_1
         3: istore_2
         4: iload_2
         5: bipush        11
         7: if_icmpge     20
        10: iload_1
        11: iload_2
        12: iadd
        13: istore_1
        14: iinc          2, 1
        17: goto          4
        20: iload_1
        21: ireturn
```

<style>
  p {
    font-size: 10px;
  }
</style>

---

JVM基于栈的指令集动画演示

<div align="center">
  <img src="/output.gif" style="width:60%"/>
</div>

8位CPU运行演示
