---
title: Java基础2
tags: 
  - Java基础
date: 2021-08-31 20:39:49
categories:
  - Java基础
---



在Java中，有两种数据类型，包括：基本数据类型和引用数据类型

## 基本数据类型

| 类型名称 | 占位  | 用法             | 范围                                                   |
| -------- | ----- | ---------------- | ------------------------------------------------------ |
| byte     | 8bit  | byte a = 100;    | -128 ~ 127                                             |
| char     | 16bit | char  b = 'a';   | 0 ~ 65535                                              |
| short    | 16bit | short c = 1;     | -32768 ~ 32767                                         |
| int      | 32bit | int d = 2;       | -2,147,483,648 ~ 2,147,483,647                         |
| long     | 64bit | long e = 3L;     | -9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807 |
| float    | 32bit | float f = 4f;    |                                                        |
| double   | 64bit | double g = 5.0d; |                                                        |
| boolean  | 1位   | true false       |                                                        |

在基本数据类型中，四则运算是普通四则运算，在Java中新增了取余运算。

前自增，后自增，前自减，后自减。

按位与，按位或，左移符，右移符

拿int来做例子吧

```java
int a = 1;
int b = 2;

public static void main(String[] args){
    System.out.println("a+b=" + a+b);
    System.out.println("a-b=" + a-b);
    System.out.println("a*b=" + a*b);
    System.out.println("a/b=" + a/b);
    System.out.println("a%b=" + a%b);
    System.out.println("a++=" + (a++));
    System.out.println("++a=" + (++a));
    System.out.println("a--=" + (a--));
    System.out.println("--a=" + (--a));
}

// 输出的结果是
a+b=3;
a-b=-1;
a*b=2;
a/b=0.5;
a%b=0;
a++=2;
++a=1;
a--=1;
--a=0;
```

输出的结果如上所示。

## 引用数据类型

类 接口 数组

### 运算符

我们知道，在数学中有四则运算--加减乘除，那么在Java中，相应的也有四则运算，但是我们要分情况讨论，因为在Java中，不仅仅有基本数据类型，还有引用数据类型。

在引用数据类型中，四则运算就没有了其他三种算法，只有加号。

例如

```java
String s = "我喜欢你";
String s1 = "呀";

public static void main(String[] args){
    System.out.println("s + s1" + s + s1);
}

// 输出的结果是
s + s1 = 我喜欢你呀
```

