---
title: SpringMVC拦截器、入参
tags: 
  - SpringMVC
date: 2022-02-14 20:08:54
categories:
  - SpringMVC
---

## 拦截器

![截图](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202203292134451.png)

拦截器封装在HandlerExcutionChain

## 调用拦截器

mappedHandler.applyPreHandle(processedRequest, response)

mappedHandler.applyPostHandle(processedRequest, response)

## 入参注入

在InvocableHandlerMethod#invokeForRequest()方法

调用getMethodArgumentValues(request, mavController, providedArgs)获取参数
