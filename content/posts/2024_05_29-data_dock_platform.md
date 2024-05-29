---
categories:
- 数据对接系列
date: '2024-05-29T08:07:00.000Z'
showToc: true
tags:
- 工作总结
- 数据对接
title: 数据对接平台

---



> 对跨系统业务数据对接流程进行标准化处理，采用配置化的方式进行数据对接需求的开发，从而提高数据对接的开发效率以及问题排查效率，减少在数据对接需求上所投入的人力成本

## 背景

随着公司业务的发展，所服务的客户越来越多时，经常会遇到需要将客户业务系统中（比如ERP等系统）的数据对接到公司所提供的系统内、或者将公司内部系统的数据推送给客户的业务系统这样的需求，这部分需求往往是提供一些基础的数据，是在系统交付上线前所必须要完成的。所以每服务一个新的客户时，一般都避免不了在这一部分投入一些开发资源。因此如果能够提高这部分需求的开发效率的话，对于系统能否尽早交付有很大的帮助。

早期公司内对于数据对接的需求处理方式最初是来一个对接需求时就定制化开发一个，后面发现对于同样的基础数据对接，其基本的对接流程都是差不多的，比如将客户业务系统的数据对接到公司内部系统的流程一般是这样的：

1. 从客户业务系统中读取数据

1. 将读取到的数据写入到数据库中间表中

1. 从中间表获取数据，并做一些转换或者其他的业务处理

1. 将处理完的数据写入到公司内部系统中

基于上面的流程，数据对接的处理做了一些优化，引入了流程处理引擎`liteflow`，将流程里的第2步和第4步做通用化处理，每次新需求开发时仅需要处理第1步和第3步的业务，减少了部分的工作量。但是这个过程中还是需要做比较多代码开发，代码质量依赖于开发个人能力，并且这个对接过程相对黑箱，一旦流程出现问题时，排查时如果缺乏了一些日志记录，很难定位到问题。

为解决以上问题，推动了数据对接平台的建设，期望平台能够满足以下核心诉求：

1. 标准化数据对接场景，沉淀公司内部平台对外的OpenApi接口

1. 针对大部分基础数据对接的需求，实现零代码开发（除标准OpenApi接口开发外），通过配置完成对接

1. 将对接流程透明化，即能够看到流程每一步的执行过程，出现问题时能够快速定位到问题

## 平台架构

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/52/7c/527cebc3c973f506da0fd0e91142f900.png)

## 核心功能

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/81/22/8122061cde4b601717c7fe2172b9df67.png)

### 站点

站点类似于租户的概念，在实际应用中一般会为一个企业建立一个站点，站点间的数据是可以相互隔离的，具体是可以通过数据库配置为每个站点配置独立的存储数据的数据库。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/09/3c/093cd4d553ae270bb87c08a3c988090a.png)

### 客户端

客户端是指部署在客户环境上的程序（具体是一个可执行jar包），由它建立与客户环境上相关数据源的联系，通过采集（读数据）或出仓（写数据）任务进行相关数据的同步。客户端在部署时需要在管理平台上进行登记，获取一个身份标识（AK访问秘钥），以此来与云端服务进行交互。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/47/09/47093d14ea7a9e42c1d485587d02855a.png)

### 数据源

数据源是对数据的输入和输出方式的抽象，具体可以是通过Web API、数据库、消息队列等方式，每种数据源都有其独特的配置，比如Web API需要指定具体的访问域名、认证方式等，数据库需要指定数据库类型（目前支持MySQL、Oracle）、连接地址、账号密码等，消息队列需要指定消息队列类型（目前支持Kafka、Solace）、连接地址、账号密码等。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/60/13/60132fbcb28dd48739bfdd438ea03d36.png)

### 接口

接口隶属于具体的数据源，是定义数据具体是从哪里输入或输出的，以及输入或输出的细节。以Web API数据源为例，一个真实的HTTP获取数据接口可以定义为一个读接口，需要指定具体的请求方式、请求头、请求体、返回结果字段映射等，接口定义完后是在后续的任务中使用。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/b9/ed/b9ed28b60a986a3580ad659d9e70b10c.png)

### 数据字典

数据字典用来管理流到云端的数据，可以是一个数据库物理表或一个常量表。通常客户端采集程序所读取并上报的数据会写到对应的数据字典表中，以供后续进行计算或出仓使用。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/cb/fe/cbfeaa5bebc6e9d6e89a8bb1bb64cc11.png)

### 采集任务

采集任务负责定义如何从外部系统采集数据到云端，这里会使用到前面所定义好的接口（具体是读接口）来采集数据，以及定义好的数据字典来存储数据，并且可以选择一个合适的采集频率执行，以此可以不断地采集新产生的数据。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/1c/94/1c94d024221be4361e9b8047a986df47.png)

### 计算任务

计算任务用于数据的转换计算，如果从客户端所采集上来的数据不满足最终需要落地的数据，可以定义计算任务来处理成最终想要的数据，目前计算任务支持定义SQL来做转换，并且是可以独立执行的，不依赖于其他任务。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/50/4b/504b81ba381d09f10456a34066e397d9.png)

### 出仓任务

出仓负责定义如何将云端的数据输出给外部系统，这里会从定义好的数据字典中读取数据，并使用定义好的接口（具体是写接口）来输出数据。出仓任务是以监控数据字典中数据状态的形式来执行的，当有未出仓的数据时会触发任务执行。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/a6/27/a6272ce57f74366cbdbcc2a6507d0dfd.png)

### 任务调度记录

任务调度记录用于记录任务（采集任务、计算任务、出仓任务）每次调度的情况，包含请求内容，响应内容等关键执行日志，便于查看或排查问题。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/22/ef/22effcf7db9d9fda6508cd426374122c.png)

### 开放接口

开放接口是⽀持外部系统主动往平台推送数据的形式，与采集任务主动采集外部系统数据的形式相反，但最终的数据都是需要落到数据字典中存储。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/a0/8d/a08d40b74c5fc6db3c8af22a323c6436.png)

### 开放接口调度记录

同任务调度记录作用，记录开放接⼝每次被调度的情况，包含请求内容，响应内容等关键执行日志，便于查看或排查问题。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/a7/d3/a7d3375e6d0fc641d26a2a90213aa553.png)

### 用户

作为一个独立的平台，有自身的用户体系，方便后续针对用户做权限控制。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/f3/84/f384e9652ac004d613d80320dae1491d.png)

### 告警

告警主要是用于当任务执⾏失败时，推送相关的告警信息到⻜书、钉钉、企微等协作平台上，以便相关责任人能够及时介入处理。

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/5d/6b/5d6b2c51aaa89207415b65532bad0476.png)

