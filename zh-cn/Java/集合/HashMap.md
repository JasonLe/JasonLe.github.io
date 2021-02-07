---
title: Java基础——对HashMap的put和resize方法进行理解
subtitle: Java基础
image: 
alt: 
date: 2020-11-22 14:55:48

caption:
  title: HashMap的put和resize方法进行理解
  subtitle: Java基础
  thumbnail: https://wimg.ruan8.com/uploadimg/image/20190402/20190402163328_31815.jpg
---
## 前言
>我们可以看到，在面试所有大厂Java岗位的时候，都有对了解HashMap原理的要求，
>意识到透彻了解HashMap的底层原理迫在眉睫，所以决定再写一篇博文对HashMap进行记忆和分享。

## 一、Put方法
<br>

开篇我们直接上干货！
<br>
### put方法源码
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

额，没什么好看的，我们继续深入putVal

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        HashMap.Node<K,V>[] tab; HashMap.Node<K,V> p; int n, i;
        
        // 首先判断现在的Hash表是不是为空，如果是空的话进行resize扩容(该方法后面会讲到)
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
            
        // 通过(n - 1) & hash计算索引
        if ((p = tab[i = (n - 1) & hash]) == null)
        
            // 如果索引位置为空，则将数据直接添加入Hash表
            tab[i] = newNode(hash, key, value, null);
        else {
        
            // 如果不为空，表示该位置已经有值，则进行下面的操作
            HashMap.Node<K,V> e; K k;
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果该位置的key值和我们要put进去的key值相同，则把对象取出来存在e变量里
                e = p;
                
            else if (p instanceof HashMap.TreeNode)
                // 这里判断如果当前位置存储的是红黑树结构，则直接进行红黑树的插入操作
                e = ((HashMap.TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            
                // 这里表示当前位置存储的是链表结构
                for (int binCount = 0; ; ++binCount) {
                
                    // 对链表进行遍历
                    if ((e = p.next) == null) {
                    
                        // 尾插法，找到链表最后一个元素，并添加到该元素后面
                        p.next = newNode(hash, key, value, null);
                        
                        // 判断添加后链表元素是否超过8，如果超过则将链表转换为红黑树结构
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        
                            // 红黑树转换操作
                            treeifyBin(tab, hash);
                        break;
                    }
                    
                    // 遍历过程判断是否存在相同的key
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            
            //如果存在key与put进入的key相同，则进行元素的替换
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 方法回调
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        
        // 判断是否达到扩容临界值，达到则进行resize扩容操作
        if (++size > threshold)
            resize();
        //方法回调
        afterNodeInsertion(evict);
        return null;
    }
```
<br>

>看到这么大一片源代码以及注释是不是头都晕了，不着急，我们再进行一波回顾和总结、


### put方法总结：
1. 首先我们调用HashMap的put方法后，需要对Hash表进行非空判断，空则使用resize方法进行扩容，非空则继续。
2. 接着通过hash计算索引位置，索引位置为空则直接进行添加，非空则继续。
3. 接着判断该位置的key值和我们要put的key值是否一样，一样则保存在变量里，以便后面进行替换操作，不一样则进行结构判断，判断该位置是红黑树结构还是链表结构，是红黑树结构则直接进行红黑树的结点添加操作，是链表则进行遍历，使用尾插法将结点添加入链表中，添加成功后进行容量判断，如果添加后容量超过既定容量8，则调用方法将链表转化为红黑树结构。
4. 最后判断Hash表的容量是否达到了扩容的临界值，如果达到了则进行resize的扩容操作。

>通过这四步，我们就完成了HashMap调用put方法进行添加元素的操作。

<br>

## 二、resize方法
<br>

还是老样子，上干货！

### resize源码
```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;

        // 记录旧表容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;

        // 记录旧的扩容临界值
        int oldThr = threshold;
        int newCap, newThr = 0;

        // 对之前旧的扩容临界值进行判断
        if (oldCap > 0) {

            // 当当前table容量大于最大值的时候返回当前table
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }

            // 把新表的长度设置为旧表长度的两倍，newCap=2*oldCap
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)

                // 左移一位，扩大两倍(还有点小押韵)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 默认初始化容量为 16
            newCap = DEFAULT_INITIAL_CAPACITY;
            // 默认初始化扩容临界值 16*0.75 = 12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            // 设置 自定义容量时的 扩容临界值
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;

        // 创建新的 table
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;

        // 如果旧数组不为空，需要将旧table里面的内容，复制到新table里面
        if (oldTab != null) {
            // 遍历整个oldtable数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    // 取到i之后，里面设置为null  防止多线程环境下循环引用 这个是对jdk1.7的一个改进
                    oldTab[j] = null;
                    // 如果就只是单单一个节点
                    if (e.next == null)
                        // 直接放到新数组位置对应的 也同样是用位运算代替，取模运算
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // //如果该节点是 树形节点 那么进行分割 作另外的处理
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 开始处理 发成冲突而形成的链表的转移
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 从表头节点e 开始循环遍历处理冲突的元素
                        do {
                            next = e.next;

                            //结果为 1：那么该元素应该放在新table的新位置
                            //结果为 0：说明该元素，放在新table的位置与旧table相同 后面会做记录
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                // 把放在新索引（原索引 + oldCap）处的元素 建立新链表
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            // 把放入原索引处的链表 插入到新table中;
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            // 把放入新索引处的链表放 插入到新的table中
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

### resize方法总结：
>resize（）方法是用来扩容的，就是当首次执行put方法，或者当添加put执行完毕后，会检查size是否大于扩容临界值，如果大于临界值，就要执行扩容操作。生成一个新的table数组，这样也就牵涉一个问题-----内容的复制。

1. 当旧的数组，只有一个元素，就是判断出它next==null，也就是说没有冲突，那就直接把该元素，该元素放置到新table里面同样索引的位置。
2. 如果要复制的节点 是一个红黑树型节点，进行红黑树操作
3. 如果要复制的节点下存在冲突，也就是有链表存在。那就从头结点开始遍历，是先通过一个巧妙的运算e.hash & oldCap，这个运算的结果，只有两种 1 和 0 。用来判断该元素，在新table中索引的位置是否发生变化。
结果是：0 直接元素存放在 newtable[ j ]
结果是：1 存放在newtable[ j+oldCap ]

<br><br><br>

> 好了，今天对于HashMap的put和resize方法的理解就到这里结束了，如果还有什么问题请在评论区告诉我！