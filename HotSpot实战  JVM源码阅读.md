#HotSpot实战  JVM源码阅读

### 宏

1. 使用宏对通用类型进行了封装

   ```c++
   #define DEF_OOP(type)
     class type##OopDesc;
     class type##Oop : public oop {
       public:
         type##Oop() : oop() {}
         type##Oop(const volatile oop& o) : oop(o) {}
         type##Oop(const void* p) : oop(p) {}
         operator type##OopDesc* () const { return (type##OopDesc*)obj(); }
         type##OopDesc* operator->() const {
           return (type##OopDesc*)obj();
         }
     }

   DEF_OOP(instantce);
   DEF_OOP(Method);
   // 只要传入不同的参数，就可以得到不同type的对应的oopDesc的定义
   ```

   hotstop同样的方式也用在了定义句柄时，只要给宏传入不同的参数买就可以得到不同的定义。

2. 很多函数都会做类似的事情，就用宏对函数进行封装
   例如返回值类型，获取当前线程，调试开关等。

3. 共同的基类
   大量的类都继承自同一个类，可以对这个类抽象成一个宏

4. 循环条件
   把常用的循环抽象成一个宏，比如需要遍历所有线程

   ```c++
   #define ALL_JAVA_THREADS(x) for (JavaThread* x = _thread_list; X; X = X->next())

   // 使用的时候可以这样
   ALL_JAVA_THREADS(p) {
     tc->do_thread(p);
   }
   // 就代表循环迭代每一条线程调试
   ```

5. 调试
   不如一些代码不应当走到的逻辑的地方，就封装一个should_not_reach_here，用于调试和输出错误信息




### 内联函数

主要是inline关键字，由于某些公用方法一直会被调用，所以没有抽象为一个方法，而是使用内连的方式，来共用方法



## Prims

JVM具有外部访问的通道，允许外部程序读取jvm内部状态信息，Prims模块定义了外部接口，主要有四个字模块组成

- JNI
  java本地接口，允许java与本地代码交互，比如与c/c++的相互调用，JNI提供的接口主要供java在运行时调用c/c++实现的库

- Perf
  sun.misc.Perf类的底层实现，用以监控虚拟机内部的Perf Data计数器

- JVM
  作为标准JNI接口的补充，用来支持一些需要访问本地哭的java api。

  - wait和notify
  - 一些函数和常量的定义，支持字节码验证和class文件格式的校验
  - 各种I/O和网络操作

- JVMTI

  虚拟机工具接口，主要是创建代理监控jvm，用于对应用程序监控调试调优，例如内存使用，CPU利用率，锁的信息等




## Runtime

JVM运行时需要很多功能，比如线程，安全点，PerfData，Stub线程，反射，VMOperation以及互斥锁等组件，均在Runtime中定义，其中包含了多个重要的模块：

- **Thread**
  维护者虚拟机内部的线程，以及java的线程

- **Arguments**
  记录传递VM参数和选项

- **StubRoutines和StubCodeGenerator**

  > 为屏蔽客户调用远程主机上的对象，必须提供某种方式来模拟本地对象，这种本地对象称为存根(stub)，存根负责接收本地方法调用,并将它们委派给各自的具体实现对象

- **Frame**
  frame表示了物理的栈桢，frame是与cpu相关的。既可以表示c栈桢也可以表示java栈桢，对于java栈桢，既可以是解释桢，也可以是编译帧。

- **CompilationPolicy**
  用来配置编译策略，Runtime中定义了两种编译策略，SimpleThresholdPolicy和AdvancedThresholdPolicy。初始化时会根据CompilationPolicyChoice来选择。

- **Init模块**
  启动时的初始化

- **VmThread**

  虚拟机启动时，会在全局创建一个单例的原生线程，该线程可以名为VM Thread。该线程的重要职责是维护一个虚拟机的操作队列，接受其他线程请求虚拟机级别的操作，如执行GC任务等。VMOperation是JVM对外对内提供的核心服务。这些操作根据阻塞类型以及是否可以进入安全点，分为四种模式

  - safepoint：阻塞，进入安全点
  - no_safepoint: 阻塞，非进入安全点
  - concurrent: 非阻塞，非进入安全点
  - async_safepoint: 非阻塞，进入安全点

- **VMOperation模块**

  虚拟机内部定义了大量的VMOperation，包括ThreadStop，ThreadDump，PrintThreads，FindDeadLocks等等。

  > HotSpot实战p46



## 启动

Launcher是用来启动JVM的工具。JRE将在三个路径下加载类

- 引导类路径（bootstrap class path）
- 已安装的扩展（installed extensions）
- 用户类路径（user class path）

java 和 javaw的启动命令中，option是传递给虚拟机的参数。分为标准和非标准两种，标准指的是将来也一定会支持的，而非标准在使用时则类似"-X"或者"-XX"

#####gamma

gamma是一个精简的Launcher



### JVM启动过程

> 由于启动的代码都是c为主的代码，在阅读源码的过程中，如果一个方法是大写字母开头，那么这个方法多半是JVM自己定义的方法，如果一个方法是小写字母开头，那么多半是系统的方法。

#### 环境变量的配置

1. main.c 166

   ```c
   return JLI_Launch(margc, margv,
                      sizeof(const_jargs) / sizeof(char *), const_jargs,
                      0, NULL,
                      VERSION_STRING,
                      DOT_VERSION,
                      (const_progname != NULL) ? const_progname : *margv,
                      (const_launcher != NULL) ? const_launcher : *margv,
                      HAS_JAVA_ARGS,
                      const_cpwildcard, const_javaw, 0);
   ```

   main.c主要根据不同的系统，对参数进行不同的处理。

2. 之后JLI_Launch方法是在java.c 216行中

3. java.c 246 InitLauncher，这个方法是在java_md_common.c中，里面又去调了JLI_SetTraceLauncher

4. JLI_SetTraceLauncher这个方法则是在jli_util.c，这个方法的目的则是确定当前jvm是不是debug模式，如果是的话就会设置_launcher_debug为True，同时记录打印开始，结束时间，一起打印当前的一些参数，重新flush standard out的流。

5. DumpState(), 就是记录一些状态，打印一些状态

6. SelectVersion，如果在启动的时候又指定版本的jre，在这个函数中会作出判断选择。jdk1.9+不支持多版本的jre。同时还会设置-Djava.awt.headless=true，就是没有显示设备、键盘鼠标的模式（大多数服务器通常都是这样的模式）。然后判断如果是jar包就要去读jarfile，如果jarfile不能读、manifest不能读或者manifest损坏了，就抛出异常。
   解析manifest用的是JLI_ParseManifest这个函数，这个函数是在parse_manifest.c 575行中，主要逻辑就是现寻找文件，如果没找到就退出并报异常。如果找到就读取参数并设置info参数，比如main_class这样的重要参数。

7. CreateExecutionEnvironment这个函数则是创建jvm虚拟环境。

   ```python
   /*  Main
    *  (incoming argv)
    *  |
    * \|/
    * CreateExecutionEnvironment
    * (determines desired data model)
    *  |
    *  |
    * \|/
    *  Have Desired Model ? --> NO --> Is Dual-Mode ? --> NO --> Exit(with error)
    *  |                                          |
    *  |                                          |
    *  |                                         \|/
    *  |                                         YES
    *  |                                          |
    *  |                                          |
    *  |                                         \|/
    *  |                                CheckJvmType
    *  |                               (removes -client, -server etc.)
    *  |                                          |
    *  |                                          |
    * \|/                                        \|/
    * YES                             Find the desired executable/library
    *  |                                          |
    *  |                                          |
    * \|/                                        \|/
    * CheckJvmType                             POINT A
    * (removes -client, -server, etc.)
    *  |
    *  |
    * \|/
    * TranslateDashJArgs...
    * (Prepare to pass args to vm)
    *  |
    *  |
    * \|/
    * ParseArguments
    * (removes -d32 and -d64 if any,
    *  processes version options,
    *  creates argument list for vm,
    *  etc.)
    *   |
    *   |
    *  \|/
    * POINT A
    *   |
    *   |
    *  \|/
    * Path is desired JRE ? YES --> Have Desired Model ? NO --> Re-exec --> Main
    *  NO                               YES --> Continue
    *   |
    *   |
    *  \|/
    * Paths have well known
    * jvm paths ?       --> NO --> Have Desired Model ? NO --> Re-exec --> Main
    *  YES                              YES --> Continue
    *   |
    *   |
    *  \|/
    *  Does libjvm.so exist
    *  in any of them ? --> NO --> Have Desired Model ? NO --> Re-exec --> Main
    *   YES                             YES --> Continue
    *   |
    *   |
    *  \|/
    * Re-exec / Spawn
    *   |
    *   |
    *  \|/
    * Main
    */
   ```

   这幅图展示了jvm虚拟环境的流程，主要起到了验证之前配置的jvm环境变量是不是正确。

8. 如果是Java Args，就解析变量，如果不是就设置默认的classpath变量。

9. LoadJavaVM中找到libjvm，把libjvm中的JNI_CreateJavaVM方法绑定到CreateJavaVM，并在后续的生成JVM时，会调用这个方法。

10. ParseArguments就是解析参数，比如-jar，--module，-m，--class-path等等

11. 之后就调用JVMInit方法，正式初始化JVM


至此，环境变量的设置和配置就完成了。接下来做虚拟机的自动和配置。



#### 初始化JVM

1. java_md_macosx.c 1040行，首先判断是不是在同一个线程中启动，-XstartOnFirstThread如果之前设置过这个参数，那么此时就会在同一个线程中启动，不然会的话就会进入ContinueInNewThread方法，一般都是进入continue这个方法。
2. ContinueInNewThread这个方法则是在java.c 2283行中，第一步会先设置栈的大小，然后继续调用ContinueInNewThread0方法。
3. ContinueInNewThread0这个方法是在java_md_macosx.c 879行。这里把JavaMain函数（java.c 386行）当作参数传到了ContinueInNewThread0方法中，在ContinueInNewThread0里面，新建并且初始化了一条线程来执行这个方法，接下去就是在新的线程中进一步去执行JavaMain函数。
4. JavaMain在java.c 386行中，主要调用了InitializeJVM，就是初始化jvm。在java.c 1458行。在设置一些变量和打印一些log之后，调用了CreateJavaVM方法，正式创建JVM




#### 创建JVM

CreateJavaVM方法在jni.cpp中，此时正式的创建JVM终于从jdk的部分跳转到了hotspot的部分，CreateJavaVM方法会根据不同的系统调用不同的方法。

1. JNI_CreateJavaVM_inner jni.cpp 3883行。这个方法内部一开始的位置有一些钩子函数，以及dtrace一些动态跟踪的接口。

2. 重点是jni.cpp 3942行，Threads::create_vm()这个方法，对虚拟机做了最终的建立工作，对应的方法是在Threads.cpp 3500行，期间调用了ThreadLocalStorage::init()，感觉是始终持有对线程的引用。Arguments::apply_ergo()这个方法主要检查了gc的组合，堆的大小，一些flag的设置，metaspace的初始化。JavaThread是定义在thread.hpp 786中的，这是非常重要的一个类。

   > 其中涉及到一个c++中的友元类，写作friend class XXXX。友元类并不属于该类，但是该类可以使用友元类的私有方法。

   init_globals是在init.cpp 100行中，在这里主要对所有全局的东西进行初始化，比如类加载器，MarkSweep（GC）等。

   初始化了ObjectMonitor，**关于ObjectMonitor具体的作用和使用的场景暂时还不清楚，但是大致上应该是一种监控对象的封装**

   > 顺便在阅读源码的过程中，顺便看到了关于栈溢出的问题。 
   >
   > 栈溢出分为三种类型，抛异常打日志的溢出、记录错误消息线程退出的溢出以及什么消息都没有的溢出
   >
   > [你假笨---栈溢出完全解读](http://www.jianshu.com/p/debef4f69a90),

3. 之后就是一些常规的记录工作，以及如果jvm创建失败的再次尝试或者抛出异常的工作。




####LoadClass

在创建jvm成功之后，java.c 1559行，这个方法jdk9和jdk8有非常大的不同之处，主要体现在GetStaticMethodID这个方法。

首先先通过GetLauncherHelperClass这个方法，找到sun/launcher/LauncherHelper这个类，调用这个类的checkAndLoadMain方法，通过sun的这个方法，就会进入到java内部的ClassLoader的逻辑。之后还对加载到的类进行校验，这部分逻辑都在LauncherHelper这个类中执行的。

如果没有找到mainclass，又会尝试去寻找applicationClass，注释上面说是个JavaFX的程序准备的，不过个人感觉没什么特别的用处

在之后就是去调用mainClass中的main方法，以及推出之后一些异常处理。





# 类机制

#### OOP-Klass二分模型

jvm并没有按照传统的，每一个java class都生成一个c++的class去对应**（c++的class长成什么样子？为什么不继承？好处是什么？）**，而是采用了一套OOP-Klass的二分模型。

![oop-klass结构图](/Users/xiwang/Documents/OpenJDK-Research/screen_shot/OOP-Klass.png)

- OOP：即普通对象指针，用来描述对象的实例信息
- Klass：Java类的c++对等体，用来描述java类

对于OOPS对象来说，主要职能是在于表示对象的实例数据，

在虚拟机内部，通过instanceOopDesc来表示Java对象，对象在内存中的布局可以分为两个部分，instantceOopDesc（或者arrayOopDesc对象头）和实例数据。对象头包括两个两个内容

- Mark Word。
  即_mark成员，存储对象运行时的记录信息，例如哈希码，gc分代年龄，锁标志等等，该成员的类型markOop。
- 元数据指针
  该指针指向Klass对象。由于Klass对象包含了实力对象所属的meta data（元数据）。所以也被称为元数据指针。虚拟机频繁的使用该指针找到位于方法区内的类型信息



instanceOopDesc和arrayOopDesc之间的区别就是，数组会多一个长度。

由于Oop对象在虚拟机运行期间会被大量的创建和频繁的使用，所以Oop中的很多方法都短小精干，并且inline，尽可能减少调用时的开销。（oop.hpp中大量的方法都有inline关键字，包括private方法）



##### 对象头

由于每创建一个java对象，都会对应的创建对象头和实力数据空间，jvm又希望在对象头重记录尽可能多的信息。因此Oop有一个flag就是使用压缩存储，即-XX:UseCompressedOops。因为在64位操作系统上，对象头的使用效率比较低下，因此在对象头重分为wideKlassOop和narrowKlassOop。如果没有开启压缩，就用wide，如果开启了压缩就用narrow。

> JDK9中没有找到wideKlassOop，虽然还是区分，但是总的来看应该是分为了oop和narrowOop



用一张图来描述虚拟机中访问对象的要点

![HotSpot图片访问机制](/Users/xiwang/Documents/OpenJDK-Research/screen_shot/jvm对象访问机制.png)



### Klass

Klass数据结构定义了所有Klass类型共享的结构和行为：描述类型自身的布局，刻画出与其他类之间的关系（父类、子类兄弟类等）。

![HotSpot图片访问机制](/Users/xiwang/Documents/OpenJDK-Research/screen_shot/Klass对象布局.png)

- _layout_helper字段
  这个字段是反应整个类对象整体布局的综合描述符。
- _name字段
  就是类名，比如说"java/lang/String"。
- _java_mirror
  表示Klass在java层的镜像类（？有点不是很理解）
- _super
  表示父类
- _subKlass
  表示子类，jvm通过_subKlass -> next_sibiling()就可以找到下个子类。



### 核心数据结构，instanceKlass

![instanceKlass](/Users/xiwang/Documents/OpenJDK-Research/screen_shot/instanceKlass内存布局.png)