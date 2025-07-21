---
title: Linux基础
date: 2022-02-14 22:35:31
tags:
  - Linux
categories:
  - Linux
---

## 1. cd

切换目录

```shell
cd /user/data/
```

## 2. ls

显示当前目录下的所有文件以及文件夹

## 3. grep

常用参数：

-a ：将 binary 文件以 text 文件的方式查找数据

-c ：计算找到‘查找字符串’的次数

-i ：忽略大小写的区别，即把大小写视为相同

-v ：反向选择，即显示出没有‘查找字符串’内容的那一行

例如：

取出文件/etc/man.config 中包含 MANPATH 的行，并把找到的关键字加上颜色

```shell
grep --color=auto 'MANPATH' /etc/man.config
```

## 4. 把 ls -l 的输出中包含字母 file（不区分大小写）的内容输出

```shell
ls -l | grep -i file
```

## 5. find

查找服务器中某个文件

```shell
find xxx.txt
```

## 6. cp

拷贝，多台机器间拷贝用 scp

```shell
cd /data/logs
scp logTest.log yim@182.129.83.121:/yim/
# 输入密码
```

## 7. mv

```shell
mv /data/logs/logTest.log /data/logsNew/
```

## 8. rm

```shell
rm -f /data/logs/logTest.log
```

## 9. ps

用于报告当前系统的进程状态

```shell
ps -ef | grep java
```

## 10. kill

杀死进程

```shell
kill -9 $pid$
```

## 11. tar

解压

```shell
tar -zxvf xxx.tar.gz
```

## 12. cat

查看文本文件内容

```shell
cat /data/logs/logTest.log
```

## 13. chmod

改变文件的权限

```shell
chmod 777 /data/logs/logTest.log
```

## 14. vim

### 基本命令

```txt
i：在当前光标位置插入文本。
x：删除当前光标所在位置的字符。
:w：保存文件。
:q：退出Vim编辑器。
:q!：强制退出Vim编辑器，不保存文件。
:wq：保存文件并退出Vim编辑器。
```

### 光标移动命令

```txt
h：将光标向左移动一个字符。
j：将光标向下移动一行。
k：将光标向上移动一行。
l：将光标向右移动一个字符。
w：将光标移动到下一个单词的开头。
e：将光标移动到当前单词的末尾。
b：将光标移动到上一个单词的开头。
0：将光标移动到当前行的开头。
$：将光标移动到当前行的末尾。
G：将光标移动到文件的末尾。
gg：将光标移动到文件的开头。
/<pattern>：向下搜索<pattern>。
```

### 文本编辑命令

```txt
dd：删除当前行。
yy：复制当前行。
p：粘贴已复制或删除的文本。
u：撤销上一次操作。
Ctrl-r：重做上一次操作。
r：替换当前光标所在位置的字符。
c：删除从当前光标位置到指定位置的文本并进入插入模式。
v：进入可视模式，选择文本。
:s/<old>/<new>/g：将当前行中的<old>替换为<new>。
:%s/<old>/<new>/g：将整个文件中的<old>替换为<new>。
```

### 插入模式命令

```txt
Esc：退出插入模式。
Ctrl-h：删除光标左侧的字符。
Ctrl-w：删除光标左侧的单词。
Ctrl-u：删除当前行的所有文本。
Ctrl-a：插入文本到行首。
Ctrl-e：插入文本到行尾。
Ctrl-t：插入一个制表符。
```

## 15. gcc

把 c 语言的源程序文件，编译成课执行程序

## 16. 从 linux 服务器上查看日志

```shell
#a.查看 catalina.out 前 200 行
cat Catalina.out | head -n 200

#b. 查看 catalina.out 倒数 200 行
cat catalina.out | tail -n 200

#c. 返回 catalina.out 中包含**的所有行
cat -n 路径/文件名 | grep 关键词

#d. 常用命令
tail -f catalina.out
tail -100 catalina.out
```
