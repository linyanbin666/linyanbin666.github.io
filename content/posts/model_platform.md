---
categories:
- 推荐工程系列
date: '2024-08-19T09:06:00.000Z'
showToc: true
tags:
- 推荐工程
- 工作总结
- 模型管理平台
title: 推荐工程-模型管理平台

---



## 背景

随着业务的发展，需要使用到模型的场景越来越多，实现一个统一的模型管理平台来规范模型处理流程，提供模型从训练、发布到预测的生命周期管理，可以极大地方便算法同学的快速迭代模型。因此，从功能角度看，要求模型管理平台至少需要具备以下能力：

- 提供模型生命周期管理，包含模型注册、发布、加载、预测、卸载

- 保障离线训练与在线预测的一致性

## 平台架构

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/5f/17/5f175178dc2a40e28cabf5d6ebd0110b.png)

- Model Admin：模型管理后台

	- Model Management：提供模型的注册、发布、模拟预测等功能

	- Transformer Management：提供预处理实现在离线机器学习平台（LAI）热加载的功能

	- File Management：提供预处理文件配置管理功能

- Model Service：模型服务

	- Model API：提供模型预测接口

		- Feature：获取特征流程

		- Preprocessing：预处理流程

		- Prediction：预测流程

	- Model Loading：提供模型加载/卸载

		- XGBoost：XGBoost模型加载

		- TensorFlow：TensorFlow模型加载

		- PMML：PMML模型格式加载

- HDFS：存放由机器学习平台（LAI）训练生成的模型相关文件

- MySQL：存放模型管理后台写入模型配置等数据

- Redis：存放用户、用户-物品交叉特征

- RocksDB：存放物品特征

## 核心功能

### 模型注册

为了方便感知线上都有哪些模型，在模型发布前，需要先在平台预先注册模型名，后续机器学习平台（LAI）的模型发布组件需要填写相关的模型名和模型配置key

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/70/83/70839df556eefc8b0d0670337a4d63d4.png)

### 模型发布

- 模型发布分成两步，首先得先通过机器学习平台（LAI）的模型发布组件进行模型发布（此时的发布只是将模型相关文件发往HDFS）

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/94/8d/948d6e796e25ac20c7421da6236c53c9.png)

- 然后再通过模型管理界面，手动点击发布按钮，往指定的环境发布模型（操作成功后模型服务会感知到并进行模型加载）

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/fa/eb/faeb1ea9eba17b73c369f5cfb88a337e.png)

### 模型预测

模型被模型服务加载后，可以通过模型管理界面的模拟请求按钮进入模拟请求界面（判断模型是否加载完毕可以通过界面上显示的日期版本（代表的是最新版本），如下图红框，未有版本加载时显示为0），填写请求相关参数即可请求模型预测出结果

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/09/17/0917ec7b4d0074ce072f9963aa26f601.png)

## 未来规划

- 下沉模型预处理到C++层，优化模型预测性能

- 模型请求监控

- 模型部署容器化隔离

