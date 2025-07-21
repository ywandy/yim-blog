---
title: 安全问题
tags: 
  - 安全
date: 2022-02-14 21:08:24
categories:
  - 安全
---

## log4j2

### 漏洞简述

在写日志的时候，通过日志参数远程调用，达到攻击者所需要的服务器或者数据库中的数据

### 主要现象

攻击的时候在logger.info("");中拼接一段idap调用的参数

ldap:rmi:1.1.1.1/username

---

## sql注入

### 漏洞简述：

在查询数据库的时候，通过拼接不安全的字符串，达到攻击者所需要的参数，如用户名，密码等

### 主要现象

字符串拼接

如：

Mybatis中的${}与#{}

```sql
#参数是 ccjj
select * from user where userName = " ccjj";

#参数是 cjj" or 1=1 
select * from user where userName = "cjj" or 1=1;
```

### 解决方案（针对登录）

1.过滤（不安全的输入过滤），或者做预编译

2.在获取用户名密码时在redis存储的token中取，redis做expireTime的限制

注：加密要放在后端加密，通过抓包没办法更改，因为不知道加密方式，直接把值改成sql注入的语录没用，传回服务器要做对应的解密

---

## XSS注入

### 漏洞简述

xss注入作为一种很常见的前端攻击方法。

### 主要现象

针对前端

```html
<!--正常输入的是chenjunjie-->
<a value="chenjunjie">chenjunjie</a>

<!--但是被攻击的时候输入 
chenjunjie" > <img src="" onerror=alert(1)><""
就变成了 -->
<a value="chenjunjie"> <img src="" onerror=alert(1)>chenjunjie"></a>
```

以上的xss攻击

### 解决方案

输出转义（标签转义） 输入过滤（正则过滤）

---

## 业务一致性

前端—传价格（抓包修改信息）

后端无做鉴权

---

## 越权

解决方案

不要相信任何前端传过来的任何信息，要从session、cockie或者redis中获取

---

## 反序列化

在传输json时，恶意传输一些能获取到关键信息的代码

---

## sdl

描述开发声明周期如何做到安全

1.需求分析 对相关的开发人员、运维做安全培训 （鉴权、传输协议、传输解密）

2.开发 代码仓库—sonar（fortify）定期扫描

3.测试 漏洞扫描web—对所有输入的参数做测试（黑盒测试）

主机漏洞扫描（扫服务器）

4.发布阶段 —对基线检查

5.上线 — 安全监控 定期扫描
