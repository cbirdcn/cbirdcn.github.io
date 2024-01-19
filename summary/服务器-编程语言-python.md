# 服务器-编程语言-python

## 语言介绍

静态语言是在编译器需要确定类型，而动态是在运行时才确定类型。

强类型是指，如果没有经过强制类型转换，不允许两种不同类型的变量相互操作。也就是说，语言层面不支持隐式类型转换。

这么说，python是动态弱类型语言。

举例
```python
a = 1
b = 2.0
c = "3"
print(a+b)
print(a+c)

'''
Hello,World
3.0
Traceback (most recent call last):
  File "script.py", line 9, in 
    print(a+c)
TypeError: unsupported operand type(s) for +: 'int' and 'str'
'''
```

python在运行时才会确定类型，也可以叫解释型语言，所以性能上不及编译型语言。但是好处是，可以做到函数级的热更新，生产力高。

python2和3版本无法兼容。

### 特点



### 基础

### 高级

#### 数据结构和算法

python内置数据结构，包括列表，集合以及字典。下面讨论查询，排序和过滤等等这些普遍存在的问题。

##### 解压可迭代对象赋值给多个变量
> 包含 N 个元素的元组或者是序列，怎样将它里面的值解压后同时赋值给 N 个变量？
> 直接赋值

```shell
>>> data = [ 'ACME', 50, 91.1, (2012, 12, 21) ]
>>> name, shares, price, date = data
>>> date
(2012, 12, 21)
>>> (year, mon, day) = data
>>> year
2012
```

如果元素的数量不匹配，会得到一个错误提示。

不仅仅只是元组或列表，只要对象是可迭代的，就可以执行分解操作。 包括字符串，文件对象，迭代器和生成器。

想丢弃一部分可以用占位符`_`

```shell
>>> data = [ 'ACME', 50, 91.1, (2012, 12, 21) ]
>>> _, shares, price, _ = data
```

##### 解压可迭代对象赋值给多个变量
> 可迭代对象的元素个数超过变量个数时，会抛出一个 ValueError。如果对象中元素个数不确定或者太多呢？
> 星号表达式

```shell
>>> record = ('Dave', 'dave@example.com', '773-555-1212', '847-555-1212')
>>> name, email, *phone_numbers = record
>>> phone_numbers
['773-555-1212', '847-555-1212']
```

哪怕解压出0个元素，也会返回列表类型

星号表达式在迭代元素为可变长元组的序列时，或分隔字符串时是很有用的

`*_`也是可以搭配使用的

##### 保留最后 N 个元素

## 算法与数据结构

### 常用算法与数据结构的python实现

### 常用的第三方包

## 编程范式

### python的面向对象

### 设计模式

### python的特殊编程方式

#### 函数式编程

## 与操作系统结合

### Linux操作系统

### 进程与线程

### 内存管理

## 网络编程

### 常用协议

#### TCP/IP

#### HTTP

### socket编程

### python并发库

## 数据持久化

### MySQL

### MongoDB

### Redis

### Memcached

### 消息队列

## web框架

### wsgi

### flask

### django

### web安全

## 系统设计

### 技术选型与成熟方案

## 主要参考

[python3-cookbook](https://python3-cookbook.readthedocs.io/zh-cn/latest/chapters/p01_data_structures_algorithms.html)