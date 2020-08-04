## 前言

对`Android`开发者来说，尤其是初中级的开发者，相信对并发编程和设计模式的了解少之又少（主要也平常用的少，学了就忘系列）。包括本人在内，这方面一直是个人进阶的软肋之一。对于并发实践经验缺乏的开发者来说，文绉绉的技术书籍和博客，会比较羞涩难懂。从本文开始，会尝试着，从不同角度和思维方向写设计模式和并发编程的知识（会比较偏理论），望能和大家共勉。

**由于无知与惰性，让我们感觉摸到了技术的天花板！**

## 面试10问

本文结合个人实际面试经验和最近学习归纳总结而出，欢迎各位大佬**点赞**支持。

通过面试10问，让大家掌握**单例模式**的**双重检查模式**和**静态内部类单例模式**，并了解其中原理。从原理进而引出本文的重点：`volatile`和`synchronized`。

### 第1问：平常在Android开发中，有用到哪么设计模式么？
当时回答：平常用的比较多的是单例模式、构造者模式、工厂模式。尤其是单例模式中双重检查模式和静态类单例模式；能够保证多线程对象唯一，不会创建多个实例导致程序执行错误或影响性能。

解读：虽然设计模式有很多种，个人来说，经常用也就单例模式了。虽然面试前突击浏览复习了，然面试一紧张，没啥卵用。所以回答一定要往自己了解的说，并引导面试官往自己会的问。
### 第2问：在纸上写一下双重检查模式和静态类单例模式代码？
心理活动：还好面试前自己已经默写过很多遍了，问题不大，哗啦啦的写出来：

**双重检查模式：**
```
public class Singleton {  
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {  
    if (singleton == null) {  
        synchronized (Singleton.class) {  
        if (singleton == null) {  
            singleton = new Singleton(); 
       		 }  
        }  
    }  
    return singleton;  
    }  
}
```
**静态内部类模式：**
```
public class Singleton {
    private volatile static Singleton singleton;

    private Singleton() { }

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
写好递给面试官：双重检查模式和单例模式都能够有效保证线程安全，又都是延时初始化，能够减少不必要的性能开销。
### 第3问：双重检查模式有什么需要注意地方？
```
public class Singleton {
    private volatile static Singleton singleton;

    private Singleton() { }  //1

    public static Singleton getSingleton() {   //2
        if (singleton == null) {  // 3.1
            synchronized (Singleton.class) {  //3.2
                if (singleton == null) {  //3.3
                    singleton = new Singleton(); //4 
                }
            }
        }
        return singleton;
    }
}
```
答：双重检查模式需要注意以下几点：
1. 构造函数得私有，禁止其他对象直接创建实例；
2. 对外提供一个静态方法，可以获取唯一的实例；
3. 即然是双重检查模式，就意味着创建实例过程会有两层检查。第一层就是最外层的判空语句：`代码3.1处的if (singleton == null)`，该判断没有加锁处理，避免第一次检查`singleton`对象非`null`时，多线程加锁和初始化操作；当前对象未创建时，通过`synchronized`关键字同步代码块，持有当前`Singleton.class`的锁，保证线程安全。
4. `Singleton`类持有的`singleton`实例引用需要`volatile`关键字修饰，因为在最后一步`singleton = new Singleton(); `创建实例的时候可能会重排序，导致`singleton`对象逸出，导致其他线程获取到一个未初始化完毕的对象。

### 第4问：刚刚讲到的第四点，为什么会有重排序，`volatile`关键字如何禁止重排序？
答：重排序是指编辑器和处理器为了优化程序性能而对指令序列进行重排序的一种手段。只要遵守`as -if-serial`语义（无论怎么重排序，单线程程序的执行结果不会改变）。所以编译器为了优化性能，可能会对下图中2和3步骤进行重排序，这种重排序时允许的，因为不会改变单线程（目前只有该线程独占该代码块）内程序的执行结果。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/481a88d9e23040eea2e041654b1a1d0d~tplv-k3u1fbpfcp-zoom-1.image)
在单线程环境是没有问题，如果在多线程环境下，程序的执行结果就会被破坏。如下图所示，线程B在第一步判空时，singleton实例的引用已经非null,所以它不进入申请锁阶段，而直接访问对象，但此对象还没初始化完成，那么对象在实际使用就会出各种问题。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6d573c1b1cb443c823b8ffebbc30322~tplv-k3u1fbpfcp-zoom-1.image)
`volatile`修饰的变量本身具有可见性和原子性，所谓的可见性是指对一个volatile变量的读值，读到的值是所有线程中最新修改的值；而原子性是指对**单个**变量的读写具有原子性。之所以会有这两个特性，是因为会在该共享变量的汇编指令之前增加`Lock`指令，该`Lock`前缀指令会在多核处理器做两件事：

1、将当前处理器缓存行的数据写回到系统内存；

2、这个写回内存的操作会使其他处理器里缓存了该内存地址的数据无效。

ps：单核处理器一时刻只能有一条线程执行，多线程是指单核CPU对不同线程进行上下文切换和调度；多核处理器同一个时刻可能多条线程（每个核一条线程）并发执行，   这时同步非常重要，现代CPU基本都是多核了。

由于volatie变量的可见性这个特性使其 **写-读** 建立起了`happens-before`关系，从内存语义的角度上说，线程A写一个`volatile`变量，实质上是线程A向接下来将要读这个`volatiel`变量的某个线程发出了通知。原理上讲的话，在写一个`volatile`变量是，JAVA内存模型（JMM）会把该线程对应的本地内存中的共享变量刷新到主内存；而在读`volatile`变量时，会把该线程对应的本地内存置为无效，从主内存中读取该变量。线程之间通过共享程序的`volatile`变量（共享状态），通过写读操作共享状态进行隐式通信。

JMM为了实现这种`volatile`内存语义，会限制编译器和处理器的部分重排序。

为编译器优化制定以下**三条规则** ：
* 第一个操作是对volatile变量的读，无论第二个操作是什么，都禁止重排序；
* 第一个操作是对volatile变量的写，第二个操作是对volatile的读，禁止重排序；
* 第二个操作是对volatile变量的写，无论第一个操作是什么，都禁止重排序；

从第2条规则就可以理解通过添加`volatile`关键字修饰单例的引用，可以禁止重排序。

根据这三条规则，编译器会在生成字节码时，在指令序列插入适当的，保守策略的内存屏障（一组CPU指令，实现对内存操作的顺序限制）。

* volatile写操作前插入StoreStore屏障；
* volatile写操作后插入StoreLoad屏障；
* volatile读操作后插入LoadLoad屏障；
* volatile读操作后插入LoadStore屏障；

以上内存屏障时非常保守，编译器在生成字节码时，也会进行部分优化，减少一些不必要的内存屏障，以提高性能。不同的处理器会根据自身的内存模型继续优化。

ps:JMM是为了屏蔽底层硬件内存模型不一致，为顶层开发提供一套标准的内存模型，让开发这专注要业务开发。
### 第5问：刚刚提到的happens-before规则，具体怎么说来的？
答：从JDK5开始，使用了新的JSR-133内存模型，该模型定义了`happens-before`
规则：
1. 程序顺序规则：一个线程中的每个操作，happens-before于该线程的任意后续操作；
2. 监视器原则：对一个锁的解锁，happens-before于随后对该锁的加锁；
3. volatile规则：对一个volatile变量的写，happens-before 于任意后续对这个volatile域的读；
4. 传递性：如果A happes-before B，B happens-before C,那么A happens-before C；
5. start()原则：线程A执行ThreadB.start()操作，start() happens-before 线程B内所有操作；
6. jion()原则：如果线程A执行 ThreadB.jion()并成功返回，那线程B的所有操作都happens-before 于A从jion()操作成功返回。
### 第6问：规则第2点讲到了锁，那锁在双重检查单例模式起了什么作用？
答：在`代码3.2处`，用到了`synchronized `关键字，对`Singletion.Class`对象进行了同步,确保了在多线程环境下只有一个线程对`Singletion`类的Class对象进行实例化。在Java中，每一个对象都可以作为锁：
1. 对于普通同步方法，锁是当前实例对象；
2. 对于静态同步方法，锁是当前类的Class对象；
3. 对于同步方法块，锁是Synchoized括号的Class对象。

### 第7问：静态内部类单例模式有没有用到锁？
答：有的，JVM在类的初始化阶段（在Class被加载后，且在线程使用之前），会执行类的初始化，JVM会去获取一个锁，这个锁能同步多个线程对同一个类的初始化。

当一个线程A获取到这个初始化锁时，其他线程想要获取初始化锁只能等待；线程A执行类静态初始化和初始化静态字段的过程，就算发生类似双重检查模式的重排序，对结果也没有影响，因为此时没有其他线程可以捕获到初始化锁。线程A初始化完毕，释放锁并通知等待获取初始化锁的线程。根据`happens-befroe`关系中的监视器规则，当其他线程获取到初始锁时，已经能看到线程A的初始化所有操作，此时静态对象已经初始化完毕，其他线程无需再初始化。

### 第8问：了解过锁的原理，知道锁存储在哪么？
答：JVM(Java虚拟机)是基于进入和退出`Monitor`对象来实现方法同步和代码块同步的。同步代码块使用`monitorenter`指令在编译后插入到同步代码块的开始位置，使用`monitorexit`插入到同步代码块的结束处或异常处，`monitorenter`必须有对应`monitorexit`指令与之配对。任何对象都有一个`monitor`与之相关联，当且一个`monitor`被持有后，将处于锁定状态。线程执行到`monitorenter`指令时，将会尝试获取对象所对应的`monitor`的所有权，即获得对象的锁。方法则是在方法的指令前增加`ACC_SYNCHRONIZED`修饰符。

`Synchronized`用的锁是存放在Java的对象头;如果对象是数组，用3字宽存储对象头，其中一字宽用于存储数组长度；非数组，则2字宽存储对象头。在32位虚拟机，1字宽=4字节=32位。



![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a442a15ae4b74e72ab8f7d6a40720bfb~tplv-k3u1fbpfcp-zoom-1.image)

### 第9问：即然了解过Java的对象头，那应该清楚锁升级的几种状态吧，说一下？
答：在Java SE6，为了减少获得锁和释放锁带来的性能消耗，引入了**偏向锁**和**轻量级锁**。意味着此时锁从低到高共有四种状态：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态。锁的状态是根据线程对锁的竞争情况来定义的。32位JVM运行状态下，Mark Work的存储结构：![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88d54b213b324b81a1acd2241a1e8f05~tplv-k3u1fbpfcp-zoom-1.image)
**偏向锁：** 线程在大多数情况下并不存在竞争条件，使用同步会消耗性能，而偏向锁是对锁的优化，可以消除同步，提升性能。当一个线程获得锁，会将对象头的锁标志位设为01,进入偏向模式.偏向锁可以在让一个线程一直持有锁，在其他线程需要竞争锁的时候，再释放锁。==》只有一个线程进入临界区。

**轻量级锁：** 当线程A获得偏向锁后，线程B进入竞争状态，需要获得线程A持有的锁，那么线程A撤销偏向锁，进入无锁状态。线程A和线程B交替进入临界区，偏向锁无法满足，膨胀到轻量级锁，锁标志位设为00。==》多个线程交替进入临界区。

**重量级锁：** 当多线程交替进入临界区，轻量级锁hold得住。但如果多个线程同时进入临界区，hold不住了，膨胀到重量级锁==》多个线程同时进入临界区。

### 第10问：为什么Synchronized够用，还要增加Volatile？
`Volatile`相对`Synchronized`来说在同步上比较轻量级，能够有效降低CPU频繁的线程上下文切换和调度。同时，`Volatile`的原子操作是针对单个`volatile`变量的写读操作，无法和`Sychronized`对整个方法或代码块起的作用相比较。



## 总结
基本每一问都会涉及到一些知识点，面试官也会从不同方向去提问，引出不同知识点。例如后面几个问题可以引出Java的内存模型，这些都是面试的高频问题。

通过本文，需要掌握`单例模式`的两种写法：[Java:单例模式我只推荐两种](https://juejin.im/post/6844903858276139021)。还需掌握`volatile`和`synchronized`的知识点。

由于个人能力有限，有错误欢迎指正；或者觉得方向不好，欢迎指导，非常感谢。

**最后希望**：`点赞`&`关注`=`true`

[我的Github](https://github.com/Android-XXM/XXM-BLOG)

本文大部分知识点来自《Java并发编程的艺术》，建议购买阅读哦。



