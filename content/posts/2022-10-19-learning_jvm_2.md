---
categories:
- 学习笔记
date: '2022-10-19T07:41:00.000Z'
showToc: true
tags:
- JVM
- Java虚拟机
title: 书籍学习-深入理解Java虚拟机（二）

---



> 本文是个人学习书籍《深入理解Java虚拟机》过程中所记录的一些笔记，内容来源于书籍

## 虚拟机字节码执行引擎

### 运行时栈帧结构

- 局部变量表：用于存放方法参数和方法内部定义的局部变量，以Slot为最小单位

- 操作数栈：用于存放方法执行过程中所做的各种运算的操作数以及结果

- 动态连接：每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接

- 方法返回地址

- 附加信息

### 方法调用

- 解析（静态）：调用目标在程序代码写好、编译器进行编译时就确定下来的

	- 静态方法（invokestatic）

	- 私有方法、实例构造器、父类方法（invokespecial）

	- final修饰的方法（invokevirtual）

- 分派（静态+动态）

	- 静态分派：依赖**静态类型**来定位方法执行版本的分派，典型应用是方法重载

	- 动态分派：运行期根据**实际类型**确定方法执行版本的分派，典型应用是方法重写

# 程序编译与代码优化

## 早期（编译器）优化

### Javac编译器

- 编译过程

	- 解析与填充符号表过程

	- 插入式注解处理器的注解处理过程

	- 分析与字节码生成过程

## 晚期（运行期）优化

### 即时编译器

- 解释器与编译器

	- 分层编译

		- 第0层，程序解释执行

		- 第1层，也称C1编译，将字节码编译为本地代码，进行简单、可靠的优化

		- 第2层（或2层以上），也称C2编译，也是将字节码编译为本地代码，但会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化

	- 热点探测

		- 基于采样的热点探测

		- 基于计数器的热点探测（HotSpot采用）

			- 方法计数器：用于统计方法被调用的次数（默认会有热度衰减，即超过一定时间限度不满足阈值条件时次数将会被减半，可见统计的不是方法的绝对调用次数）

			- 回边计数器：用于统计一个方法中循环体代码执行的次数（默认没有热度衰减，因此统计的是该方法循环执行的绝对次数）

- 编译优化技术

	- 公用子表达式消除

		- 如果表达式E已经计算过了，且到现在E中所有变量的值没有发生变化，那么E的这次出现就成为了公共子表达式

	- 数组范围检查消除

	- 方法内联

	- 逃逸分析

		- 为优化代码手段提供依据的分析技术，基本行为就是分析对象动态作用域

		- 如果能证明一个对象不会逃逸到方法或线程之外，可为这个变量进行一些高效的优化

			- 栈上分配

			- 同步消除

			- 标量替换

# 高效并发

## Java内存模型与线程

### Java内存模型

- 目的

	- 屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果

- 主内存和工作内存

	- 所有共享变量都存储在主内存，每条线程都有自己的工作内存

	- 工作内存中保存了该线程使用到的变量的主内存副本拷贝

	- 线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量

- 内存间交互操作

	- lock：作用于主内存的变量，把一个变量标识为线程独占状态

	- unlock：作用于主内存的变量，把一个处于锁定状态的变量释放出来

	- read：作用于主内存的变量，把一个变量的值从主内存传输到线程的工作内存中，以便随后的load操作使用

	- load：作用于工作内存的变量，把read操作从主内存中得到的变量值放入工作内存的变量副本中

	- use：作用于工作内存的变量，把工作内存中一个变量的值传递给执行引擎

	- assign：作用于工作内存的变量，把一个从执行引擎接收到的值赋给工作内存的变量

	- store：作用于工作内存的变量，把工作内存中一个变量的值传输到主内存中，以便随后的write操作使用

	- write：作用于主内存的变量，把store操作从工作内存中得到的变量的值放入主内存的变量中

- volatile特性/语义

	- 保证变量对所有线程的可见性，即当线程修改了变量的值后，新值对于其他线程来说是立即得知的

	- 禁止指令重排序优化

- 先行发生原则（用来确定一个访问在并发环境下是否安全）

	- 程序次序规则

		- **在一个线程内**，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作

	- 管程锁定的规则

		- 一个unlock操作先行发生于后面对同一个锁的lock操作

	- volatile变量规则

		- 对一个volatile变量的写操作先行发生于后面对这个变量的读操作

	- 线程启动规则

		- Thread对象的start方法先行发生于此线程的每个动作

	- 线程终止规则

		- 线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行

	- 线程中断规则

		- 对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测到是否有中断发生

	- 对象终结规则

		- 一个对象的初始化完成先行发生于它的finalize()方法的开始

	- 传递性

## Java与线程

### Java线程调度

- 抢占式线程调度

### 线程的实现

- 使用内核线程的实现

- 使用用户线程的实现

- 使用用户线程加轻量级进程混合实现

- Java线程的实现：基于操作系统原生线程模型来实现

### 状态转换

- 线程状态

	- 新建（New）：创建后未启动的线程处于这种状态

	- 运行（Runable）：Runable包括了操作系统线程状态中的Running和Ready

	- 无限等待（Waiting）：处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显示地唤醒

		- 没有设置Timeout参数的Object.wait()方法

		- 没有设置Timeout参数的Thread.join()方法

		- LockSupport.park()方法

	- 限期等待（Timed Waiting）：处于这种状态的线程不会被分配CPU执行时间，不过无须等待被其他线程显示地唤醒，在一定时间之后它们会由系统自动唤醒

		- Thread.sleep()方法

		- 设置了Timeout参数的Object.wait()方法

		- 设置了Timeout参数的Thread.join()方法

		- LockSupport.parkNanos()方法

		- LockSupport.parkUntil()方法

	- 阻塞（Blocked）：线程被阻塞了

	- 终止（Terminated）：已终止线程的状态

## 线程安全与锁优化

### Java中的线程安全

- 不可变

- 绝对线程安全

- 相对线程安全

	- 保证对这个对象单独的操作式线程安全的，不需要做额外的保障措施，但对于一些特定顺序的连续调用，可能需要在调用端使用额外的同步手段来保证

- 线程兼容

	- 对象本身并不是线程安全的，但可以通过在调用端正确地使用同步手段来保证对象在并发环境中可以安全地使用

- 线程对立

	- 无论调用端是否采取了同步措施，都无法在多线程环境中并发使用的代码

### 线程安全的实现方法

- 互斥同步

	- synchronized

	- ReentrantLock

- 非阻塞同步

	- CAS

- 无同步

	- 可重入代码：如果一个方法，它的返回结果是可以预测的，只要输入了相同的数据，就都能返回相同的结果

	- 线程本地存储

		- ThreadLocal

## 锁优化

### 自旋锁与自适应自旋

- 自旋锁：线程在获取不到锁时不进行阻塞，而是执行一个忙循环（自旋），等待锁的释放

- 自适应自旋：自旋时间不固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者状态来决定

### 锁消除

- 锁消除：虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除，主要判断依据来源于逃逸分析的数据支持

### 锁粗化

- 锁粗化：如果一系列的连续操作都对同一对象反复加锁和解锁，虚拟机会扩展锁的范围，以减少频繁地进行互斥同步操作导致不必要的性能损耗

### 轻量级锁

- 目的：在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗

- 加解锁过程

	- 加锁

		- 在当前线程栈帧中创建锁记录（Lock Record）的空间，用于存储锁对象目前Mark Word的拷贝

		- 使用CAS尝试将对象的Mark Word更新为指向Lock Record的指针，操作成功即线程获得该对象的锁，操作失败再检测Mark Word是否已指向当前线程的栈帧，如果是则说明当前线程已经拥有了这个对象的锁，否则说明锁被其他线程抢占了，要膨胀为重量级锁

	- 解锁

		- 使用CAS操作把对象当前的Mark Word和线程中复制的Displaced Mark Word替换回来，如果替换成功，整个同步过程就完成了。如果替换失败，说明有其他线程尝试过获取该锁，那就要在释放锁的同时，唤醒被挂起的线程

### 偏向锁

- 目的：消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能

- 偏向过程

	- 当锁对象第一次被线程获取时，虚拟机会把对象头标志设为偏向模式，同时使用CAS操作把线程ID记录到对象的Mark Word中，如果操作成功，则持有偏向锁的线程以后进入这个锁相关的同步块时，可以不进行任何同步操作

	- 当有另外一个线程尝试获取这个锁时，偏向模式就宣告结束
