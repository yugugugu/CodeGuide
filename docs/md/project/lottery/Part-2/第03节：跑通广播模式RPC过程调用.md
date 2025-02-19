---
title: 第03节：跑通广播模式RPC过程调用
pay: https://t.zsxq.com/Ia6AUvj
---

# 第03节：跑通广播模式RPC过程调用

作者：小傅哥
<br/>博客：[https://bugstack.cn](https://bugstack.cn)

>沉淀、分享、成长，让自己和他人都能有所收获！

- 分支：[210804_xfg_buildFramework](https://gitcode.net/KnowledgePlanet/Lottery/-/tree/210804_xfg_buildFramework)
- 描述：构建工程完成RPC接口的实现和调用

当基础的工程模块创建完成以后，还需要给整个工程注入`灵魂`，就是让它可以跑通。这个过程包括一个简单的 RPC 接口功能实现和测试调用，那么这里为了让功能体现出一个完整度，还会创建出一个库表在 RPC 调用的时候查询出库表中的数据并🔙返回结果。那么在这个分支上我们就先来完成这样一个内容的实现。

## 零、优秀作业

- [整合dubbo远程调用rpc测试的时候配置一开始没改导致报错 @卡布奇诺](https://t.zsxq.com/06JaiyFIm)
- [被广播模式坑了下，放上我最后调通的配置 @蛋蛋🏃₄₂.₁₉₅ *](https://t.zsxq.com/06F2V3FqJ)
- [跑通广播模式RPC过程调用 @一点江南](https://t.zsxq.com/06uB27eIu)
- [跑通广播模式RPC过程调用 @numqin](https://t.zsxq.com/06R3rBEA2)
- [RPC 问题排查：Failed to configure a DataSource: 'url' attribute is not specified @sky是清新色](https://t.zsxq.com/06uRvJema)
- [问题排查：由于我本地的mysql是8.0版本，项目的jdbc版本较低，导致项目运行报错 @有生之年有幸相见](https://t.zsxq.com/06vbUNNbe)
- [问题排查：在项目根目录install时出现“Unable to find main class”编译错误 @404](https://t.zsxq.com/06FaqVneM)
- [问题排查：Error creating bean with name 'cn.itedus.lottery.test.ApiTest' @远航](https://t.zsxq.com/06rj6E6QZ)
- [跑通广播模式RPC过程调用 @Geroge Liu](https://t.zsxq.com/06B2NFMBm)
- [跑通广播模式RPC过程调用 @一行。](https://t.zsxq.com/06rjuNFYR)
- [抽奖系统第3-5打卡学习 @CCAT](https://t.zsxq.com/06VRNZfe2)
- [RPC终于跑通了；启动类加注解、@Reference直连、禁用掉了虚拟网络 @YanL99](https://t.zsxq.com/06EIMR7ee)
- [DDD + RPC 各个分层模块的 POM 配置和依赖关系 @Jachin](https://t.zsxq.com/07EqJqRrN)
- [跑通广播模式RPC过程调用，JDK版本问题 @Cc](https://t.zsxq.com/0cJf5EQIc)
- [第一个问题是扫描不到相对应的bean @A](https://t.zsxq.com/0c9V7T8PT)
- [前三节的学习，下面是详细的步骤，给自己记录也给大家一点帮助。@Yu](https://t.zsxq.com/0etx1mgu2)
- [使用dubbo跑通RPC调用，完整操作步骤流程记录 @夜空的寂静](https://t.zsxq.com/0eh7ysSr6)

## 一、创建抽奖活动表

在抽奖活动的设计和开发过程中，会涉及到的表信息包括：活动表、奖品表、策略表、规则表、用户参与表、中奖信息表等，这些都会在我们随着开发抽奖的过程中不断的添加出来这些表的创建。

那么目前我们为了先把程序跑通，可以先简单的创建出一个活动表，用于实现系统对数据库的CRUD操作，也就可以被RPC接口调用。在后面陆续实现的过程中可能会有一些不断优化和调整的点，用于满足系统对需求功能的实现。

**活动表(activity)**

```sql
CREATE TABLE `activity` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增ID',
  `activity_id` bigint(20) NOT NULL COMMENT '活动ID',
  `activity_name` varchar(64) CHARACTER SET utf8mb4 DEFAULT NULL COMMENT '活动名称',
  `activity_desc` varchar(128) CHARACTER SET utf8mb4 DEFAULT NULL COMMENT '活动描述',
  `begin_date_time` datetime DEFAULT NULL COMMENT '开始时间',
  `end_date_time` datetime DEFAULT NULL COMMENT '结束时间',
  `stock_count` int(11) DEFAULT NULL COMMENT '库存',
  `take_count` int(11) DEFAULT NULL COMMENT '每人可参与次数',
  `state` tinyint(2) DEFAULT NULL COMMENT '活动状态：1编辑、2提审、3撤审、4通过、5运行(审核通过后worker扫描状态)、6拒绝、7关闭、8开启',
  `creator` varchar(64) CHARACTER SET utf8mb4 DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_activity_id` (`activity_id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='活动配置';
```

- 活动表：是一个用于配置抽奖活动的总表，用于存放活动信息，包括：ID、名称、描述、时间、库存、参与次数等。

## 二、POM 文件配置

按照现有工程的结构模块分层，包括：
- lottery-application，应用层，引用：`domain`
- lottery-common，通用包，引用：`无`
- lottery-domain，领域层，引用：`infrastructure`
- lottery-infrastructure，基础层，引用：`无`
- lottery-interfaces，接口层，引用：`application`、`rpc`
- lottery-rpc，RPC接口定义层，引用：`common`

在此分层结构和依赖引用下，各层级模块不能循环依赖，同时 `lottery-interfaces` 作为系统的 war 包工程，在构建工程时候需要依赖于 POM 中配置的相关信息。那这里就需要注意下，作为 Lottery 工程下的主 pom.xml 需要完成对 SpringBoot 父文件的依赖，此外还需要定义一些用于其他模块可以引入的配置信息，比如：jdk版本、编码方式等。而其他层在依赖于工程总 pom.xml 后还需要配置自己的信息。
