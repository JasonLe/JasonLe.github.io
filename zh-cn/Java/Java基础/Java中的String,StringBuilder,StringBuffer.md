# Java中的String，StringBuilder，StringBuffer
<br>

## 简要介绍

>#### 一、三者区别
 + String对象是常量，它的值不能被创建后改变，StringBuffer和StringBuilder可以可变；
 + StringBuilder非线程安全（单线程使用），String与StringBuffer线程安全（多线程使用）；
 + 如果程序不是多线程的，那么使用StringBuilder效率高于StringBuffer。
 
 <br>
 
>#### 二、String 类——String字符串常量
 + 字符串广泛应用 在Java 编程中，在 Java 中字符串属于对象，Java 提供了 String 类来创建和操作字符串。
 + 需要注意的是，String的值是不可变的，这就导致每次对String的操作都会生成新的String对象，这样不仅效率低下，而且大量浪费有限的内存空间。
 + 所以为了应对经常性的字符串相关的操作，引入了两个新的类——StringBuffer类和StringBuild类来对此种变化字符串进行处理。

<br>

>#### 三、StringBuffer 和 StringBuilder 类——StringBuffer字符串变量、StringBuilder字符串变量

 - 当对字符串进行修改的时候，需要使用 StringBuffer 和 StringBuilder 类。
 - 和 String 类不同的是，StringBuffer 和 StringBuilder类的对象能够被多次的修改，并且不产生新的未使用对象。
 - StringBuilder 类在 Java 5 中被提出，它和 StringBuffer 之间的最大不同在于 StringBuilder的方法不是线程安全的（不能同步访问）。
 - 由于 StringBuilder 相较于 StringBuffer有速度优势，所以多数情况下建议使用 StringBuilder 类。然而在应用程序要求线程安全的情况下，则必须使用StringBuffer 类。

<br>

## 具体解答
>#### 一、为什么说String对象不可变，StringBuffer和StringBuilder对象可变呢？
这里我们就要介绍，其实String对象的底层实际上是一个char[ ]数组：
![String的底层](https://img-blog.csdnimg.cn/20200813134902799.png)
用final修饰的
+ 如果引用为基本数据类型，则该引用为常量，该值无法修改；
+ 如果引用为引用数据类型，比如对象、数组，则该对象、数组本身可以修改，但指向该对象或数组的地址的引用不能修改。

综上所述，value指向不可以改变，但是value[ ]数组的值可以变，再加上private关键字对其进行封装达到value[ ]值也不可以改变的目的。
***
而我们看一下StringBuffer和StringBuilder，它们分别都继承自AbstractStringBuilder抽象类，而AbstractStringBuilder抽象类中没有使用final定义，所以是可变的。
![StringBuffer&StringBuilder底层](https://img-blog.csdnimg.cn/20200813141914715.png)
<br>
>#### 二、为什么说StringBuilder是非线程安全的，String和StringBuffer都是线程安全的呢？

 - String中的对象是不可变的，也就可以理解为常量，所以线程安全。
 - StringBuffer对方法加了同步锁或者对调用的方法加了同步锁，所以是线程安全的。
 - StringBuilder并没有对方法进行加同步锁，所以是非线程安全的。
<br>
>#### 三、为什么说String效率低下，而StringBuilder和StringBuffer效率高呢？

每次对String对象进行改变的时候，都会生成一个新的String对象，然后将指针指向新的String对象。

而StringBuffer每次都会对StringBuffer对象本身进行操作，而不是生成新的对象并改变对象引用。

所以相同情况下使用StringBuilder相比使用StringBuffer仅能提升10%-15%左右的性能提升，但却要冒多线程不安全的风险。
<br>
>#### 最后总结

 1. 如果要操作少量的数据用 String；
 2. 多线程操作字符串缓冲区下操作大量数据 StringBuffer；
 3. 单线程操作字符串缓冲区下操作大量数据 StringBuilder。
