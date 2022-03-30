---
categories:
- 推荐工程系列
date: '2022-03-30T04:48:00.000Z'
showToc: true
tags:
- 推荐工程
- 工作总结
- 实验平台
title: 推荐工程-实验平台

---



## 背景

早期的AB实验一般都是单层实验，逻辑实现简单，比如直接对用户流量进行分组，即可假设所有用户固定分桶区间为[0, 99]，每个实验组为区间内的一个不重叠子集，然后使用userId/deviceId计算hash值对分组数取模，得出用户所命中的实验组。这种方法易用且具有相对的灵活性，但会存在以下的问题：

- 扩展性差，只能同时支持少量实验。虽然可以将流量分成很多组，但实验组一般都会要求样本数不能太少，这对本身流量较少的业务来说，分组的数量会受到限制。

- 流量饥饿问题：总流量固定，如果某些实验组需要比较大的流量进行实验，则需要减少其他实验组的流量，导致其他实验组只有少量流量甚至没有流量可用。

- 流量偏置问题：比如某个实验把大部分男性用户都获取了，导致其他实验很少甚至没有男性用户的样本，实验结果会有偏差。

为解决以上问题，需要搭建一个能支持大量线上实验且能够保障独立实验间不会相互影响的平台，通过调研发现，Google发布的论文【Google 重叠实验框架：更多，更好，更快地实验】（[译文地址](https://github.com/oldratlee/translations/tree/master/overlapping-experiment-infrastructure-more-better-faster-experimentation)）已经为如何实现这样的一个平台提供了相关实践，因此我们基于该论文以及自身业务的特点搭建了分层实验平台，使用该平台替代了旧的单层实验方式，大大的提升了做线上实验的效率。

## 平台架构

![%E5%AE%9E%E9%AA%8C%E5%B9%B3%E5%8F%B0.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/5b/40/5b40cb4e8567bc5b1df7a2ffb375adf1.png)

平台架构

平台整体架构比较简单，主要以下三层结构：

### **存储层**

该层只依赖MySQL数据库，用来存放平台所生产的相关数据。

### **接口层**

接口层包含对外提供服务的Http接口以及语言相关的SDK包，应用可以通过以上两种方式进行接入，此外，平台也开放了相关写入API，方便接入方通过编码的方式生成场景或实验的相关配置。

### **展示层**

展示层提供面向用户的界面功能，包含应用管理、场景管理、参数管理、实验记录、实验效果和白名单管理功能，基本覆盖日常实验所需功能。

## 核心功能

### 应用管理

应用是平台用来管理不同项目（可能是一个服务或多个服务）的接入信息所抽象出来的概念，每个接入的项目需要创建对应的应用，不同应用的数据是隔离的（没有应用的权限则看不到应用数据），负责人或管理员可以将应用权限授权给其他用户。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/57/98/5798b6490c9af346e5e147888d6760ca.png)

应用列表页

### 场景管理

场景是归属于应用内的数据，对应于实际线上需要进行实验的某个场景，比如猜你喜欢推荐场景等，这里的场景也是Google论文里面所提到的默认域的实现。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/17/15/1715779c3a135f7680255d750c08285c.png)

场景列表页

进入到某个场景的内部，就是实际的编辑实验的位置，这里我们对于论文里面所提到的域、层、发布层以及实验都做了实现，但也根据自身业务做了一些精简，比如限制了层内不能再嵌套域、参数只能在默认域（默认参数）、实验（实验参数）和发布层（灰度参数）存在。

树形图的编辑方式，可快速构建层级结构或修改单一元素：

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/62/2a/622a89328689299df5753f617d9bb40d.png)

场景编辑页（树形图结构）

目录树的编辑方式，可同时对多个元素进行修改：

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/01/ef/01ef6aaf63b7c98c1252a40c86f070d6.png)

场景编辑页（目录树结构）

另外，当实验越来越多时所需要的参数也随之变多，如果每次都在实验上直接修改，久而久之可能会忘记实验参数是来源于哪个实验内容，对此我们弄了针对实验参数单独管理的功能，通过独立出实验配置参数，可以追溯每个实验内容所涉及到的参数。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/51/25/51255e355b0bec050af2e32664cc57a1.png)

实验参数列表页

当配置好实验参数并且测试/预发环境对实验组进行验证后，可以选择发布到线上的环境，每次发布（包含上线、下线、全量）实验会记录对应的实验内容，一个是方便产品或相关开发知道线上存在哪些实验，另一个是后续实验效果的统计会基于这些记录进行。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/e1/2a/e12a9ab7a5741d571d1289b6146cc60f.png)

实验记录列表页

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/c0/dc/c0dcdea08c4695eed4b9721634277867.png)

实验效果页

### 白名单管理

白名单是用来强制指定某个用户所命中的实验组，不需要经过hash分组过程，常用来测试或验证实验组所修改的东西。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/f5/53/f553c44af35e949b9392f588a5a084ba.png)

白名单列表页

## 展望未来

目前平台已经满足主要业务的实验需求，核心功能已逐渐成熟稳定，平台所得到的反馈也更多的是在接入和使用流程方面，因此未来可以考虑在用户使用体验、平台周边工具这方面进行迭代优化，可以是以下的几个点：

- 提供通用的场景配置模板，提高场景配置效率

- 提升实验效果统计数据的实时性（目前一般是T+1计算）

- 提供流量评估相关的工具等

