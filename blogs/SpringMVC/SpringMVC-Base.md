---
title: SpringMVC原理分析
tags: 
  - SpringMVC
date: 2022-02-14 20:08:54
categories:
  - SpringMVC
---

## 总括

- 前端控制器（DispatcherServlet）：接收请求，响应结果，相当于电脑的CPU。

- 处理器映射器（HandlerMapping）：根据URL去查找处理器

- 处理器（Handler）：（需要程序员去写代码处理逻辑的）

- 处理器适配器（HandlerAdapter）：会把处理器包装成适配器，这样就可以支持多种类型的处理器，类比笔记本的适配器（适配器模式的应用）

- 视图解析器（ViewResovler）：进行视图解析，多返回的字符串，进行处理，可以解析成对应的页面

---

## SpringMVC执行流程

1. 用户发起请求到前端控制器
2. 前端控制器请求处理器映射器（HandlerMappering）去查找处理器（Handle）：通过xml配置或者注解进行查找
3. 找到以后处理器映射器（HandlerMappering）像前端控制器返回执行链（HandlerExecutionChain）
4. 前端控制器（DispatcherServlet）调用处理器适配器（HandlerAdapter）去执行处理器（Handler）
5. 处理器适配器去执行Handler
6. Handler执行完给处理器适配器返回ModelAndView
7. 处理器适配器向前端控制器返回ModelAndView
8. 前端控制器请求视图解析器（ViewResolver）去进行视图解析
9. 视图解析器像前端控制器返回View
10. 前端控制器对视图进行渲染
11. 前端控制器向用户响应结果

---

## 配置mvc

### 1.新建一个spring文件

### 2.配置bean

#### 2.1.声明处理器映射器

#### 2.2.声明处理器适配器

#### 2.3.声明视图解析器

视图解析器拼接视图的前缀与后缀

---

## 前端控制器（DispatcherServlet）

继承了Servlet，走的是doService方法，调用的是父类（FrameworkServlet）

### 1.请求进来DispatcherSevlet

### 2.doDispatch(request, response);

doDispatch内部有获取处理器的方法getHandler()，获取到的是HandlerExecutionChain类型的数据

### 3.遍历HandlerMapping

发现BeanNameUrlHandlerMapping，返回mappedHandler

### 4.根据Handler获取handlerAdapter

使用getHandleAdpter(mappedHandler.getHandler)，返回HandlerAdapter

springmvc会根据不同的handler返回不同的适配器（返回simpleControllerHandler）

### 5.调用handler

### 6&7.返回modelAndView

使用ha.handle()，把handler转成了controller（实现了controller接口）

执行modelAndView中的方法

### 8.处理结果集

processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)

### 9.返回view对象

执行resolveViewName(viewName, mv.getModelIntenal(), locale, request);

### 10.把view的数据填充进去

view.render(mv.getModelInteral(), request, response);

### 11.输出

renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);

rd.forward(request, response);

---

### 注解方式

根据Handler获取handlerAdapter

（返回RequestMappingHandlerAdapter）

返回modelAndView的时候使用

AbstractHandlerMethodAdapter

doInvoke()的getBridgedMethod().invoke(getBean(), args)

getBridgedMethod() 就是拿到原调用的方法

getBean() 就是拿到原调用方法的bean

---

实现HttpRequestHandler

重写handleRequest方法

直接重定向到指定页

使用的是BeanNameUrlHandlerMapping

HttpRequestHandlerAdapter实际上就是把过来的转成对应的handler，返回mv对象

把handler转成HttpRequestHandler

---

装载配置的时候

调用的是dispatcherSevlet#initStategies()

BeanFactoryUtils.beansOfTypeIncludingAncestors();

工厂模式装配组件到上下文（ApplicationContext）中

就是说初始化的时候就已经把工程内所有的方法用到的组件映射关系给罗列好了

![image-20240318100037255](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202403181000346.png)
