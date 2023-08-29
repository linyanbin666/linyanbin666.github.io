---
categories:
- 采坑系列
date: '2023-08-29T16:31:00.000Z'
showToc: true
tags:
- Mybatis-Plus
- 逻辑删除
title: 采坑系列-Mybatis-plus 3.0-RELEASE逻辑删除Bug

---



> 采坑系列是记录日常学习/工作中所遇到的问题，可能是一个Bug、一次性能优化、一次思考等，目的是记录自己所处理过的问题，以及解决问题这一过程中所做的思考或总结，避免后续再犯相似的错误。

# 问题描述

公司后端开发规范中需要有逻辑删除字段来实现软删除（即针对删除操作不会删除实际的记录，只会更新对应逻辑删除字段的值，并且在查询时自动加上过滤条件过滤掉被标识为删除的记录），项目中所使用的ORM框架为`Mybatis-plus`，其支持实现逻辑删除的功能，详情见[官方文档](https://baomidou.gitee.io/mybatis-plus-doc/#/logic-delete)。但在实际使用过程中发现一个奇怪的现象：前辈们告知**逻辑删除字段需要作为实体类的最后一个属性（即放在最后），否则其功能会失效**，具体原因不是很清晰。

因个人在开发中将逻辑删除字段放在了基类中，不确定会不会影响逻辑删除功能的正常使用，本着严谨以及好奇的态度，决定实际探究一下：为什么逻辑删除字段需要作为实体类的最后一个属性？（保持质疑-。-）

# 探究原因

## 复现

首先清楚项目中所使用的`Mybatis-plus`版本是`3.0-RELEASE`（这个很关键，因为就是它的问题！）

然后就是通过Demo（详细见[示例代码](https://github.com/linyanbin666/samples/tree/main/mybatis-plus-logic-delete-bug)中的`DemoEntityMapperTest#testSelect`）复现出查询时逻辑删除功能生效与失效的现象（尝试将`BaseEntity`中的逻辑删除字段`deleted`放在最后和非最后，查看效果）

- 当`deleted`放在最后时，从生成的SQL语句看是有自动添加逻辑删除条件的，如下图：

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/6c/7f/6c7f6d1063db191527e20d33aabb0b6c.png)

- 当`deleted`放在非最后时（例如移到`modifyTime`上面），可以看到并没有自动加上逻辑删除条件，如下图：

![Untitled.png](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/cd/57/cd57d964653dc5bb53bbd35103f6004c.png)

## 调试

### 确定入口

以上用于复现问题使用的方法为`BaseMapper#selectList`，根据<u>*[Mybatis-plus的BaseMapper实现原理](https://juejin.cn/post/7002423698565103653)*</u>，自定义逻辑的实现会对应一个`AbstractMethod`实现类，又因为我们Demo中使用了`LogicSqlInjector`，其提供了一系列的实现类，而`selectList`方法对应的实现类为`LogicSelectList`，所以从该类进入调试

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/d4/1d/d41d8cd98f00b204e9800998ecf8427e)

### 跟踪执行

跟踪`LogicSelectList`类的执行可看到逻辑删除过滤条件的处理是放在父类`AbstractLogicMethod`的`sqlWhereEntityWrapper`方法中，其直接获取`TableInfo`类的`logicDelete`属性，问题中所返回的结果为`false`，跳过了逻辑删除过滤条件的处理。到这里可以知道为什么逻辑删除功能没有生效了，但是具体原因还得看看为什么`logicDelete`属性是设置为`false`的

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/d4/1d/d41d8cd98f00b204e9800998ecf8427e)

### 寻找根因

通过寻找对`TableInfo`类的`logicDelete`属性进行设置的地方（通过IDEA Find Usages），最终可以找到是在`TableFieldInfo`类中的构造函数中进行设置的，其中`initLogicDelete`方法就是判断字段是否有标识`@TableLogic`注解

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/d4/1d/d41d8cd98f00b204e9800998ecf8427e)

而`TableFieldInfo`实例创建的地方是在`TableInfoHelper`类的`initTableFields`方法中。该方法会通过反射获取实体类中的所有字段（包含父类的），然后遍历这些字段创建成对应的`TableFieldInfo`元数据对象，即每个字段会调用一次`TableFieldInfo`的构造函数，而在构造函数中会对`TableInfo`类的`logicDelete`属性直接进行**覆盖赋值**（参考上图）。从这里就可以看到，如果最后一个字段不是逻辑删除字段的话，`TableInfo`类的`logicDelete`属性就为`false`了

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/d4/1d/d41d8cd98f00b204e9800998ecf8427e)

到这里就知道为什么逻辑删除字段需要作为实体类的最后一个属性了，根本原因就是`Mybatis-plus` `3.0-RELEASE`内部代码的问题，正常的话应该只需要判断有一个字段标识了逻辑删除后，后面就不应该再对`logicDelete`属性进行赋值了。另外从获取实体类字段列表的方法中可以看到，返回的列表会先添加实体类自身的字段，再添加父类中非重名的字段（以此类推）。因此**如果实体类有父类的话，逻辑删除字段必须放在最顶级的父类中（Object之下），并且作为最后一个属性**。

![](https://raw.githubusercontent.com/linyanbin666/pic/master/notionimg/d4/1d/d41d8cd98f00b204e9800998ecf8427e)

# 结论

通过以上的分析，可以回答最初提到的问题：

- Q：为什么逻辑删除字段需要作为实体类的最后一个属性？

- A：是因为项目所使用的`Mybatis-plus 3.0-RELEASE`版本中有Bug所导致，其需要保证通过反射所获取的实体类字段列表中，逻辑删除字段（标识了`@TableLogic`注解的）是最后一个元素。

- Q：将逻辑删除字段放在了基类中，会不会影响逻辑删除功能的正常使用？

- A：如果实体类定义了基类，逻辑删除字段必须放在基类中，同样需作为最后一个属性，逻辑删除功能才能正常使用

回答了最初的问题后，需要另外注意的是`Mybatis-plus`中是通过**反射**来获取实体类字段列表的（即`Class#getDeclaredFields`方法），我们实际上依赖的是这个方法返回的字段顺序，需要看该方法能否保证字段返回的顺序和我们在类中所定义的顺序一致。从该方法的注释上我们可以看到以下注释：

> The elements in the returned array are not sorted and are not in any particular order.

	返回数组中的元素没有排序，也没有任何特定的顺序。

也就是说该方法并不会保证字段返回的顺序和我们在类中所定义的顺序一致，原因可见*[Java反射中的getDeclaredFields()方法的疑问？](https://www.zhihu.com/question/52856385)*（不保证并不代表就会不一致，至少目前项目在用的环境能够一致-。-），因此在`Mybatis-plus 3.0-RELEASE`版本中即使我们保证将逻辑删除字段作为实体类的最后一个属性定义，还是存在逻辑删除功能失效的风险。如果要彻底避免这个风险，那只能升级`Mybatis-plus`的版本（`Mybatis-plus 3.0.1`版本开始已修复上述提到的Bug），但这个要结合实际情况评估升级的成本与必要性，不过对于新的项目来说，建议还是使用新版本。

# 参考

1. [MyBatis-Plus的BaseMapper实现原理](https://juejin.cn/post/7002423698565103653)

1. [Java反射中的getDeclaredFields()方法的疑问？](https://www.zhihu.com/question/52856385)

