---
categories:
- 采坑系列
date: '2022-03-03T15:27:00.000Z'
showToc: true
tags:
- ArrayList
- 并发问题
title: 采坑系列-ArrayList并发写出现Null值

---



> 采坑系列是记录日常学习/工作中所遇到的问题，可能是一个Bug、一次性能优化、一次思考等，目的是记录自己所处理过的问题，以及解决问题这一过程中所做的思考或总结，避免后续再犯相似的错误。

## 问题描述

在并发场景下ArrayList是线程非安全的，并发往ArrayList里面添加元素，可能导致内部出现Null值的情况，尽管你添加进去的元素能保证非Null，但Null值不是来源你添加进去的元素，而是因为并发add导致ArrayList内部索引错乱，下面例子可复现错误（如果跑一次没有复现需跑多几次）：

```java
public static void main(String[] args) throws Exception {
    List<Integer> arr = new ArrayList<>();
    int threadNum = 10;
    CountDownLatch await = new CountDownLatch(threadNum);
    CyclicBarrier barrier = new CyclicBarrier(threadNum);
    for (int i = 0; i < threadNum; i++) {
      int finalI = i;
      new Thread(() -> {
        try {
          barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
          e.printStackTrace();
        }
        arr.add(finalI);
        await.countDown();
      }).start();
    }
    await.await();
    System.out.println("arr.size=" + arr.size());
    for (int i = 0; i < arr.size(); i++) {
      if (arr.get(i) == null) {
        System.out.println("Value is null, index=" + i);
      }
    }
  }
```

复现错误的输出如下，可见index为0的位置出现了Null值：

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/25/21/25212518d205dbb58594b27b9af24688.png)

Debug断点后可看到如下arr变量的数据，发现在index为0的位置确实是Null值，而且在arr里面缺少了本应该存在的数值0，说明在并发情况下ArrayList发生了不可预知的错误。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/ca/2a/ca2aa8b4ba5003550f184074ef19c4b6.png)

## 探究原因

### 初探

查看ArrayList的add方法源码，内部操作很简单，只有三行代码：

1. `ensureCapacityInternal`，内部对`modCount`值加1，并判断添加元素是否需要扩容，需要则对底层数组进行扩容

1. `elementData[size++] = e`，将待添加元素e存放进数组，然后执行size++

1. `return true`，返回添加成功

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

从上面可看到修改ArrayList元素的地方是第二行代码`elementData[size++] = e;`，该操作包含了`size++`还有赋值操作，我们知道自增操作是非原子性的，可拆分为：读-加-写，那么这一语句就包含了4个操作：读-加-写-数组元素赋值，类似于下面的操作（下面的代码只是做说明，实际执行的是字节码指令）：

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/5f/89/5f8967a85b981d4ec64c96896cdf7a05.png)

拆分出上面几个操作后，我们可以尝试分析下在多个线程同时执行上面操作时，会出现哪些问题

1. **覆盖/丢失值**

- 线程1拿到的size为0，执行完操作2后让出CPU给到线程2，此时index为0，size未更新还是0

- 线程2开始执行拿到的size为0，index为0，执行完以上所有操作后让出CPU给到线程1，此时size被更新为1

- 线程1继续执行步骤3和4，index为之前获取到的值0，size被更新为1，因此出现线程1的元素覆盖了线程2的元素（线程2的元素丢失了）

到这里貌似没能找出为啥会出现Null值的原因，反而分析出了另外一个问题-。-；既然Null值问题不是出现在第二行代码，那么就只有可能在第一行代码里面出现了（废话-__-）

### 再探

回头看`ensureCapacityInternal`方法里面做了什么事情：

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
		// 如果是空数组，则返回DEFAULT_CAPACITY（默认是10）与要求的最少容量minCapacity中的最大者
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 默认扩大原数组长度的1/2倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 不满足所要求的容量则以要求的容量为准
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果新容量大于期望的最大容量，确认最终使用的容量（MAX_ARRAY_SIZE or Integer.MAX_VALUE)
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

- `calculateCapacity`：计算数组所需要的容量

- `ensureExplicitCapacity`：修改`modCount`值，确认是否数组所需要的容量大于现有数组的长度，是则表明需要对数组进行扩容

- `grow`：对数组进行扩容，默认扩大原数组长度的1/2倍，如果扩大后还不满足所要求的容量`minCapacity`，则以要求的容量为准，如果新容量`newCapacity`大于`MAX_ARRAY_SIZE`，则取`MAX_ARRAY_SIZE`或`Integer.MAX_VALUE`（取决于`minCapacity`是否大于`MAX_ARRAY_SIZE`）

PS：`MAX_ARRAY_SIZE`值为`Integer.MAX_VALUE - 8`，为啥减8可自行查看`MAX_ARRAY_SIZE`的Java doc

从上面代码来看，会修改数组的只有扩容方法里面的这一句：`elementData = Arrays.copyOf(elementData, newCapacity)`，这里会将原数组数据复制到扩大长度后的新数组，并将数组引用指向新数组，考虑以下的并发情况：

- 线程1拿到的size为0，要插入元素需要进行扩容，还未执行`elementData = Arrays.copyOf(elementData, newCapacity);`时让出了CPU给到线程2

- 线程2拿到的size也为0（因线程1还未执行到`elementData[size++] = e;`），要插入元素需要进行扩容，扩容完成后往下标为0的位置设置了元素并成功返回，此时`size`变成了1

- 线程1继续执行扩容，拿到的数组可能还是旧的`elementDate`（根据JVM的内存模型，每个线程在使用主内存中的变量时，会将变量拷贝到自己的工作内存中，线程1读取`elementDate`时可能线程2对`elementDate`的修改还未写回主内存，所以拿到的是旧值），线程1扩容后的新数组（数据全部为null）将赋值给`elementDate`，之后线程1将执行`elementData[size++] = e;`，此时获取到的`size`可能为1（由线程1对`size`的修改被线程2读到了），即会给下标为1的位置赋值，然后`size`变成了2，至此导致下标为0的位置出现了null值

## 解决方法

如果必须在并发情况下使用List，可按以下方式解决List并发安全问题，可根据实际业务场景选用：

- 使用线程安全的`java.util.Vector`类（具体是**对每个方法**加`synchronized`保证线程安全，不推荐使用）

- 使用`java.util.Collections`类的`synchronizedList`方法对List进行包装，让其变成线程安全的类（具体是**对非线程安全的方法**内部加了`synchronized`保证线程安全，推荐使用）

- 使用`JUC`的`java.util.concurrent.CopyOnWriteArrayList`类（写操作加锁且数组变量用`volatile`修饰，读操作无锁，具体是在写内部使用`ReentrantLock`加锁，并且采用复制原数组-往新数组添加元素-替换旧数组的形式，保证读操作的线程安全，适合读多写少的场景）

## 总结

本次针对ArrayList并发写出现Null值的问题进行探究，最初怀疑是因为`size++`不是原子性导致的（而且貌似网上也挺多这样说的-_-），按照思路分析下去后才发现问题不是出现在这里，不过也发现了`size++`不是原子性导致的另外一个并发问题（覆盖/丢失值）；后面回过头来分析扩容操作，看到针对实例变量的赋值后，有点想不起一个线程对实例变量修改后，另一线程用到这一实例变量会怎么样了，又去看了下JVM内存模型相关的资料，最后才确认了问题的根源。现在想想一个小问题涉及到的知识点其实蛮多的，做下复盘其实也是一次很好的学习机会。



