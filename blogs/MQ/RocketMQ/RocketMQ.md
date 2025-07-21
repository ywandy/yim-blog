---
title: RocketMQ
tags:
  - MQ
date: 2022-02-14 22:35:05
catogeries:	
  - MQ
---

## 前言

参照官方文档

**https://github.com/apache/rocketmq/tree/master/docs/cn**

消息消费顺序是指定顺序，同一个topic遵循FIFO（first in first out 先进先出）原则，不同topic之间可以并行消费

---

## RocketMQ基本原理

Broker推送Topic到NameSrv

Producer推送Topic到Broker

Consumer从Broker中获取对应的Topic进行消费

NameSrv定时发送心跳包检测alive的Broker，对非活动的Broker进行下线处理，定时清除

![部署方式](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202202152147748.png)

---

## 发送消息的方式

### 1.同步发送

### 2.异步发送

### 3.单向发送（多用于日志发送）

---

## Topic跟tag的区别

topic是一级菜单，tag是二级菜单。

eg：不同物流平台的订单是不同的topic，同一平台的不同模块的订单是不同的tag，不同topic之间的tag是完全隔离的。

#### 到底什么时候该用 Topic，什么时候该用 Tag？

##### 1、消息类型是否一致

如普通消息，事务消息，定时消息，顺序消息，不同的消息类型使用不同的 Topic，无法通过 Tag 进行区分。

##### 2、业务是否相关联

没有直接关联的消息，如淘宝交易消息，京东物流消息使用不同的 Topic 进行区分；而同样是天猫交易消息，电器类订单、女装类订单、化妆品类订单的消息可以用 Tag 进行区分。

##### 3、消息优先级是否一致

如同样是物流消息，盒马必须小时内送达，天猫超市 24 小时内送达，淘宝物流则相对会会慢一些，不同优先级的消息用不同的 Topic 进行区分。

##### 4、消息量级是否相当

有些业务消息虽然量小但是实时性要求高，如果跟某些万亿量级的消息使用同一个 Topic，则有可能会因为过长的等待时间而"饿死"，此时需要将不同量级的消息进行拆分，使用不同的 Topic。

