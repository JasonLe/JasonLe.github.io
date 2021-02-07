---
title: ArrayList和LinkedList的区别
subtitle: ArrayList和LinkedList的区别
image: 
alt: 
date: 2020-09-09 10:18:28

caption:
  title: ArrayList和LinkedList的区别
  subtitle: ArrayList和LinkedList的区别
  thumbnail: https://wimg.ruan8.com/uploadimg/image/20190402/20190402163328_31815.jpg
---
# 写在前面
>众所周知，在面试Java岗位的时候，面试官最常问的问题之一就是，你常用哪些Java集合框架啊？可以说一说ArrayList和LinkedList之间有什么区别吗？  

>我们今天就来聊一聊到底有什么样的区别。

<br>

## 不同点
<br>

### ArrayList
 - ArrayList的底层实现是基于动态数组的数据结构，所以它可以以O(1)的时间复杂度去对元素进行随机访问
 - 因为底层是数组，所以对于插入和删除操作来说，最坏则需要遍历整个数组，而且在插入元素时候(除了使用尾插法)还需要重新计算数组大小，更新索引，并且在数组空间满了的时候还必须将数据重新装入一个扩容的新数组。
 
### LinkedList
 
 - LinkedList是基于双向链表的数据结构，每一个节点储存三个值，  每一个元素都和前后两个元素连接在一起，所以LinkedList比ArrayList更占内存
    1. 前节点(pre)
    2. 后节点(next)
    3. 值(item)  
 - 底层是链表实现，所以不能实现随机访问，如果要查找一个节点的数据必须进行遍历，时间复杂度为O(n)
 - 相对于ArrayList，LinkedList对增加和删除操作速度更快，因为当元素被添加到集合任意位置的时候，不需要像数组那样重新计算大小或者是更新索引
>在添加或删除数据的时候，ArrayList经常需要复制数据到新的数组，而LinkedList只需改变节点之间的引用关系，这就是LinkedList在添加和删除数据的时候通常比ArrayList要快的原因

<br>

## 那么什么时候使用ArrayList什么时候使用LinkedList呢？

- ArrayList更适合存储和获取数据。 LinkedList更适合操纵数据。
- 当你的应用不会随机访问数据的时候，更适合LinkedList 。因为如果你需要LinkedList中的第n个元素的时候，你需要从第一个元素顺序数到第n个数据，然后读取数据。
- 你的应用更多的插入和删除元素，更少的读取数据时候使用 。因为插入和删除元素不涉及重排数据，所以它要比ArrayList要快。
> ### 总结：
>换句话说，ArrayList的实现用的是数组，LinkedList是基于链表，ArrayList适合查找，LinkedList适合增删