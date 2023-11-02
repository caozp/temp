DVM是Google专门为Android平台开发的虚拟机。



## DVM与JVM的区别

DVM不是一个JVM，DVM并没有遵循JVM规范来实现

###### 基于架构不同

JVM基于栈，则意味着需要去栈中读取数据，所需的指令更多

DVM基于寄存器。基于寄存器实现的虚拟机虽然在硬件、通用性上要差一些，但是它在代码的执行效率上更胜一筹

###### 执行的字节码不同

JVM会通过响应的class文件和jar文件获取相应的字节码

DVM会用dx工具将所有的class文件转换为一个dex文件，然后DVM会从dex文件读取指令和数据。执行顺序为

> java文件-->class文件-->dex文件

在apk文件中，只包含一个dex文件，这个dex文件将所有的class里面所包含的信息全部整合在一起，这样再加载就加快了速度。class文件存在很多的冗余信息，dx工具会去除冗余信息，并把所有的class文件整合到dex文件中，减少了IO操作，加快了类的查找速度。



###### DVM运行在有限的内存中同时运行多个进程

每个应用都运行在一个DVM实例中，独立的进程可以防止虚拟机崩溃时，所有的程序都关闭。



###### DVM由Zygote创建和初始化

每当系统创建一个应用程序时，Zygote就会fork自身。所有的DVM实例都会和Zygote共享一个内存区域，节省了内存开销



###### DVM有共享机制

不同应用之间在运行时可以共享相同的类，拥有更高的效率。



### DVM的运行时堆

DVM采用**标记清除**算法进行GC，它有两个space，Zygote Space和Allocation Space。

Zygote Space用来管理Zygote进程在启动过程中预加载和创建的各种对象。

Zygote Space不会触发GC，在Zygote进程和应用程序进程之间会共享Zygote Space。

在Zygote进程fork第一个子进程之前，会把Zygote Space分为两部分，原来那部分堆依旧叫Zygote Space，未使用的那部分堆叫Allocation Space。以后的对象都会在Allocation Space分配和释放。



###### GC日志

GC日志会打印到logcat中，具体格式为

> dalvikvm：<GC_Reason> <Amount_freed> <Heap_stats> <External_memory_stats> <Pause_time>



GC reason就是引起GC的原因，有以下几种

- GC_CONCURRENT:当堆开始填充时，并发GC可以释放内存
- GC_FOR_MALLOC:堆内存已满
- GC_HPROF_DUMP_HEAP:当请求创建HPROF文件来分析堆内存时出现的GC
- GC_EXPLICIT:显式的GC，例如System.gc()
- GC_EXTERNAL_ALLOC:仅使用与API小于等于10





###### ART和DVM的区别

DVM中的应用每次运行时，字节码都需要进行JIT编译器编译为机器码，这使得应用程序的运行效率降低。

而ART中，系统在安装应用程序时会进行预编译AOT，将字节码预先编译成机器码并存储在本地，这样应用程序每次运行时就不需要执行编译了。