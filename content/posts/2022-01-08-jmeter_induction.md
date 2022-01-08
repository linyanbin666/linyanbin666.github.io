---
categories:
- 工具
date: '2022-01-08T09:57:00.000Z'
showToc: true
tags:
- 工具
- 压测
- JMeter
title: JMeter压测工具使用入门

---



## 安装

### JMeter安装

JMeter的安装很简单，直接访问 [官方下载页面](https://jmeter.apache.org/download_jmeter.cgi) 下载即可。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/3c/bb/3cbb309951eb3a6450eb8ff22762db20.png)

### **JMeter Plugins Manager安装**

JMeter的插件管理安装不是必须的，不过安装插件管理后可以下载一些有用的插件，建议也安装一下。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/76/b5/76b5d5d31a78b61da10e95c8202496e3.png)

启动JMeter后，可以通过`Options `→ `Plugins Manager `选项打开插件管理界面。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/95/8a/958a3c41feb082d5d46a3bafb3ec3a46.png)

插件管理界面可以在`Installed Plugins`栏看到已安装的插件，如果将已安装的插件去掉勾选，则表示要卸载插件；在`Available Plugins`栏可以看到可用的插件，可以勾选需要安装的插件，然后点击右下方的` Apply Changes and Restart JMeter `按钮保存即可。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/55/4f/554fb57f54476d9bf976a156b3dd8c77.png)

## 启动

JMeter的启动也很简单，只需要找到安装目录下的 `bin/jmeter.bat`（Windows系统）或 `bin/jmeter.sh` （Mac或Linux系统）执行即可，这种方式或启动JMeter的GUI界面。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/7f/66/7f66d2f375d9f6d3b2cb8e2208e0ead9.png)

## 使用

下面以配置固定吞吐量10/s来测试HTTP接口为例说明如何使用。

### 配置线程组

首先需要在测试计划上点击右键 `Add` → `Threads(Users)` → `Thread Group` 添加线程组，线程组用来模拟访问测试接口的用户。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/bf/c4/bfc429f9ab2cd97800e6909ff2e226e8.png)

线程组涉及了如下的一些配置：

- **Action to be taken after a Sampler error**：由于采样器失败或断言失败而导致执行遇到错误时该如何处理

  - **Continue**：忽略错误，继续执行

  - **Start Next Thread Loop**：忽略错误，终止当前线程的循环，执行下一个循环

  - **Stop Thread**：终止当前执行的线程，不影响其他线程的执行

  - **Stop Test**：在当前正在执行的线程执行完后停止测试

  - **Stop Test Now**：立即终止当前正在执行的线程后停止测试

- **Number of Threads(users)**：定义执行时想模拟的线程（用户）数量

- **Ramp-up period(seconds)**：全部线程启动执行的时间，比如线程数量设置为10，启动执行时间设置为100秒，那么第一个线程将在第0秒（测试执行开始时）启动执行，然后每个线程将在10秒(100/10)后启动执行

- **Loop Count**：每个线程的执行次数，可以设置为不限次数（Infinite）或固定次数，比如循环数设置为2，线程数为100，那么总的将运行200次

- **Same user on each iteration**：在每次循环中使用相同的用户，当有设置HTTP Cookie Manager时，开始这个选项后会将在第一个响应中获得的cookie用于后续的请求

- **Delay Thread creation until needed**：延迟创建线程所需的资源，比如设置要启动1000个线程，即使设置了Ramp-up，JMeter也会立即为所有线程分配内存；启用该选项则可以在新线程开始执行时分配内存

- **Specify Thread lifetime**

  - **Duration(seconds)**：设置线程组运行多长时间，跟线程数和每个线程运行次数配合使用，即使线程数和线程运行次数还未跑完，持续时间到了，线程组直接停止

  - **Startup delay(seconds)**：设置线程启动延时时间，是指多少秒后才开始启动线程，不是说每个线程都延迟多少秒启动

### 配置HTTP采样器

在线程组上点击右键 `Add` → `Sampler`→`Http Request`添加HTTP采样器，一般测试GET方法或简单的POST方法基本上用这个采样器就足够了，但有时候我们需要设置一些请求头信息，可以看到在HTTP采样器这个里面是没有办法配置的，这时候就需要用到配置元素HTTP Header Manager。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/ef/13/ef134dfe1ecb0066051e7f3dcacd397d.png)

### 配置HTTP头管理器

在线程组上点击右键 `Add` → `Sampler`→`HTTP Header Manager`添加HTTP请求头管理器，点击下方的`Add`可以添加请求头配置项，如下添加了Content-Type: application/json。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/59/2a/592abf2088ec2daf860448056dd04d37.png)

### 配置固定吞吐量计时器

在线程组上点击右键 `Add` → `Timer`→`Constant Throughput Timer`添加固定吞吐量计时器，该计时器可以控制请求执行的吞吐量，下面设置了线程组内的所有线程执行吞吐量控制在600/min（10/s）左右（可能会有误差），当然这个的前提是要线程组的执行次数能满足要求。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/e9/80/e9806446d2fc7e009f31442093bd649d.png)

### 配置聚合报告

在线程组上点击右键 `Add` → `Listener`→`Aggregate Report`添加聚合报告，聚合报告可以查看样本执行的耗时、吞吐量和接发数据量情况，如下是控制吞吐量在10/s左右执行1000次的聚合结果。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/5d/64/5d644d5c75d2b8eed45777c311155732.png)

聚合报告中每一行（除了最后一行）为每个HTTP采样器的执行情况，最后一行为统计所有采样器的执行情况，每一列表示的意思如下说明：

- **Label**：HTTP采样器设置的名称

- **#Samples**：总请求样本数

- **Average**：平均耗时，单位ms

- **Median**：中位数耗时，单位ms

- **90%Line**：90分位线耗时，单位ms

- **95%Line**：95分位线耗时，单位ms

- **99%Line**：99分位线耗时，单位ms

- **Min**：最小耗时，单位ms

- **Maximum**：最大耗时，单位ms

- **Error%**：错误百分比

- **Throughput**：每秒处理的请求数（n/sec）

- **Received KB/sec**：每秒接收的数据量（KB/sec）

- **Sent KB/sec**：每秒发送的数据量（KB/sec）

## Linux下执行JMeter

在本地配置好要执行的测试计划后保存成脚本xxx.jmx（自由命名），然后将文件上传到Linux服务器上，然后输入以下命令执行（以下~/jmeter/apache-jmeter-5.4.1为服务器上JMeter的安装目录）。

```shell
~/jmeter/apache-jmeter-5.4.1/bin/jmeter -n -t ~/jmeter/xxx.jmx -l ~/jmeter/xxx.jtl
```

JMeter启动常用参数说明：

```plain text
-h 帮助：打印出有用的信息并退出
-n 非 GUI 模式：在非 GUI 模式下运行 JMeter
-t 测试文件：要运行的 JMeter 测试脚本文件
-l 输出文件：记录结果的文件
-r 远程执行：启动远程服务
-H 代理主机：设置 JMeter 使用的代理主机
-P 代理端口：设置 JMeter 使用的代理主机的端口号
```

测试计划执行完成后会输出xxx.jtl文件，将其下载到本地，然后启动JMeter，添加所需的监听器并打开浏览即可。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/4f/56/4f567190df427695d491d21cd16007ce.png)

## 总结

本文首先介绍了如何安装JMeter，安装步骤很简单，直接上官网下载安装包安装即可；接着通过一个简单的HTTP请求接口测试例子介绍了如何使用JMeter，并在其中对一些关键的配置进行说明；最后介绍了如何在Linux上运行JMeter，通过非GUI模式启动脚本运行并输出执行结果文件，再通过GUI界面打开文件浏览即可。

## 参考

1. **[5 must know features of Thread Group in Jmeter](http://www.testingjournals.com/5-must-know-features-thread-group-jmeter/)**

1. **[Guide to JMeter Thread Groups](https://www.redline13.com/blog/2018/05/guide-jmeter-thread-groups/)**

1. **[Introducing JMeter 5.2!](https://www.blazemeter.com/blog/introducing-jmeter-5.2)**

1. **[How to Use the Delay Thread Creation on JMeter](https://www.blazemeter.com/blog/how-to-use-the-delay-thread-creation-on-jmeter)**

1. **[Understand and Analyze Aggregate Report in Jmeter](http://www.testingjournals.com/understand-aggregate-report-jmeter/)**

1. **[linux环境运行jmeter并生成报告](https://www.cnblogs.com/imyalost/p/9808079.html)**



