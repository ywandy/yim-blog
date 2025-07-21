---
title: 自己总结的知识点
tags: 总结
date: 2022-02-14 20:08:54
---

1.声明变量后，必须显式进行初始化，否则值为null
2.static finall声明为类常量，可以在一个类中多个方法被调用
3.一个数的平方根，使用Math.sqrt()方法
4.强转规则：
5.在get获取前端传入的数据时，前端ajax的url必须带参数传入，必须要是表单包裹input框。ajax方法使用document.getParameter(name)获取input框中输入的数值。后端controller的参数写到括号中，或者用HttpServletRequest.getParameter(name)获取前端传入的数值
6.使用equal()方法时，常量写到外面，变量写到里面，防止报错
7.for和增强for的区别：for有下标，foreach没有；foreach更简洁，无需使用索引进行遍历，底层用的是iterater进行遍历
8.while和do while的区别：while是先判断再执行，do while是先执行再判断
9.判断字符串是否为空--StringUtils.isEmpty()
10.String的一些方法
- replace(a,b)--a为旧字符串，b为新字符串
- trim()--去掉头部和尾部的空格
- toLowerCase()--将大写字母改成小写
- toUpperCase()--将小写字母改成大写
11.StringBuilder的一些方法
- append()--在字符串后加一些字节或字符，并返回新的字符串

12.面向对象到底是个啥

​		我们知道，java的特性就是面向对象（Object Oriented Programming），首先他与面向过程不同，面向过程是将一个程序拆分为多个过程，比如说开始做啥，接着做啥，最后做啥。而面向对象是将要做的东西封装成一个个的类，然后操纵这些类，达到实现程序的效果，而这些类也被称作为对象。
​		封装：形式上来看，封装只不过是将数据和行为组合在同一个包中，并对对象的使用者隐藏了数据的实现方式。对象中的数据成为实例域，操纵数据的过程称为方法。对于每个特定的类实例对象都有一组特定的实例域值，这些值的集合就是这个对象的当前状态。无论何时，只要向对象发动一个消息，它的状态就有可能发生改变。比如说：一个人有名字，年龄，性别等属性。这些属性在java中声明的时候会通过private关键字不给外部类使用。
​		继承：子类复用父类的方法就叫继承，子类可以使用父类的方法，但是父类不能使用子类的方法。注：继承与实现的区别，继承可以增加自己的方法，而实现只能是实现父类的方法，不能自行实现。
​		多态：一件事物的多种不同的表现形式。比如说：动物有猫，狗等。

13.类之间的关系：

​		依赖：如果一个类操纵另一个类的对象，我们就说一个类依赖另一个类
​		聚合：类a的对象包含着类b的对象
​		继承：如果类a扩展类b，类a不但包含从类b继承的方法，还会拥有一些额外的功能。

14.关于初始化变量

如果将一个方法用于一个值为null的对象上，那么会产生runtimeException

局部变量不会自动地初始化为null，而必须通过调用new或将他们设置成为null进行初始化

new操作的返回值也是一个引用

15.mysql的mybatis模糊检索优雅写法

使用INSTR(字段名, #{})=0

代替like concat(#{},'%')

16.在循环删除map的数据时，要用iterator

参照 https://www.cnblogs.com/zhuyeshen/p/10956822.html

```java
Iterator<Map.Entry<String, List<String>>> iterator = kdNewMap.entrySet().iterator();        					    	while(iterator.hasNext()){            
	List<String> data = new ArrayList<>();
  data.add(iterator.next().getKey());            
  data.add(getCycle().substring(0, 6));            
  data.add(BILLINGCYCLE.substring(0, 6));            
  SqlRowSet sqlRowSet2 = dcDao.queryForRowset(ISEXIST, data.toArray());            
  if (sqlRowSet2.next()) { 
    iterator.remove();            
  }        
}   
```

==不能用==

```java
for (Map.Entry<String, List<String>> kdDatas : kdMap.entrySet()) {            
	List<String> datas = kdDatas.getValue();            
  for (int i = 0; i < datas.size(); i++) {                
    List<String> data = new ArrayList<>();                
    String accNum = kdDatas.getKey();                
    data.add(accNum);                
    data.add(getCycle());                
    data.add(BILLINGCYCLE);                
    List list = dcDao.queryForSQL(ISEXIST, data.toArray());                
    if (!list.isEmpty()) {
      kdMap.remove(kdDatas.getKey());                
    }            
  }        
}        
```

会报java.util.ConcurrentModificationException

![截图](https://pic-1304161434.cos.ap-guangzhou.myqcloud.com/img/202202142029143.png)
