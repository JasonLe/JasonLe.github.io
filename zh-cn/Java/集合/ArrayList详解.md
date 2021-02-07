---
title: ArrayList源码分析
subtitle: ArrayList
image: 
alt: 
date: 2021-01-20 19:18:45

caption:
  title: ArrayList源码分析
  subtitle: ArrayList
  thumbnail: https://wimg.ruan8.com/uploadimg/image/20190402/20190402163328_31815.jpg
---

# 一、ArrayList  
  
## 1、ArrayList介绍
<br>

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{}
```

ArrayList是实现List接口的动态数组，之所以说是动态，是因为它的数组大小是可变的。每个ArrayList都有一个容量  <br>

```java
   /**
   * Default initial capacity.
   */
   private static final int DEFAULT_CAPACITY = 10;
```

+ 容量是指用来存储列表元素的数组的大小。默认初始容量为10。
+ 随着ArrayList中元素的增加，它的容量也会不断的自动增长。在每次添加新的元素时，ArrayList都会检查是否需要进行扩容操作，扩容操作带来数据向新数组的重新拷贝，所以如果我们知道具体业务数据量，在构造ArrayList时可以给ArrayList指定一个初始容量，这样就会减少扩容时数据的拷贝问题。
+ 当然在添加大量元素前，应用程序也可以使用ensureCapacity（该方法的作用是预先设置Arraylist的大小）操作来增加ArrayList实例的容量，这可以减少递增式再分配的数量。

**注意，ArrayList实现不是同步的**。如果多个线程同时访问一个ArrayList实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。所以为了保证同步，最好的办法是在创建时完成，以防止意外对列表进行不同步的访问：

```java
List list = Collections.synchronizedList(new ArrayList(...)); 
```
<br>

## 2、ArrayList源码分析    
<br>

### 2.1 ArrayList底层使用List
```java
transient Object[] elementData;
```
transient关键字是什么意思呢？如果用transient关键字声明一个变量，那么代表着它的值不需要维持，当一个对象被序列化的时候，transient声明的变量的值不包括在序列化的表示中，然而非transient声明的变量是被包括进去的。

**PS: Java中的序列化是能够将一个实例对象的状态信息写入到一个字节流中使其可以通过socket进行传输、或者持久化到存储数据库或文件系统中；然后在需要的时候通过字节流中的信息来重构一个相同的对象。**

也许会有读者有疑问，那么这样一来ArrayList就不可以序列化了吗？
那么请看下面的分析
我们发现在ArrayList中有这样两个方法 writeObject() 和 readObject()

```java
     /**
     * 将 ArrayList 实例的状态保存到流（即序列化它）。
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // 写出元素计数和任何隐藏的东西
        int expectedModCount = modCount;
        s.defaultWriteObject(); // 写出当前类的所有非静态字段(non-static)和非瞬态字段(non-transient)到ObjectOutputStream

        // 将size写出到ObjectOutputStream
        s.writeInt(size);

        // 因为ArrayList是可扩容的,在添加元素时,可能会扩容,这个时候会存在一些没有使用的空间,所以采用这种方式,来节约空间和减少序列化的时间
        for (int i=0; i<size; i++) {　　// size代表数组中储存的元素的个数
            s.writeObject(elementData[i]); // 有序的将elementData中已使用的元素读出到流中
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

     /**
     * 从流中重构 ArrayList 实例（即，对其进行反序列化）。
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // 读入size, 和所有隐藏的东西
        s.defaultReadObject();

        // 读入容量
        s.readInt(); // ignored

        if (size > 0) {
            // 就像clone（）一样，根据大小而不是容量来分配数组
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // 按正确的顺序读入所有元素。
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

<br>

### 2.2 ArrayList的构造方法
ArrayList提供了三个构造方法： 
1. ArrayList()：默认构造函数，提供初始容量为10的空列表。
2. ArrayList(int initialCapacity)：构造一个具有指定初始容量的空列表。
3. ArrayList(Collection<? extends E> c)：构造一个包含指定 collection 的元素的列表，这些元素是按照该 collection 的迭代器返回它们的顺序排列的。

```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
        //
    }

    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        }
    }

    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
<br>

### 2.3 ArrayList的新增操作  
ArrayList提供了五个方法来实现数组的增加
1. add(E e)
2. add(int index, E element)
3. addAll(Collection<? extends E> c)
4. addAll(int index, Collection<? extends E> c)
5. set(int index, E element)  

其中用到的方法：
1. public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)。它的根本目的就是进行数组元素的复制。即从指定源数组中复制一个数组，复制从指定的位置开始，到目标数组的指定位置结束。将源数组src从srcPos位置开始复制到dest数组中，复制长度为length，数据从dest的destPos位置开始粘贴。

```java
    /**
     * 在数组最后添加元素
    */
    public boolean add(E e) { 
        ensureCapacityInternal(size + 1);  //先做扩容检测，如果容量不够就扩容。
        elementData[size++] = e; //将列表末尾元素指向e。
        return true;
    }

    /**
     * 将指定的元素插入此列表中的指定位置。
    */
    public void add(int index, E element) {
        rangeCheckForAdd(index);  //判断index索引是否正确
        ensureCapacityInternal(size + 1);   //扩容检测
        /**
        * System.arraycopy对源数组进行复制处理
        * 主要是从index+1位置开始，向右复制移动size-index个元素
        */
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        elementData[index] = element;
        size++;
    }

    /**
    * 按照指定 collection 的迭代器所返回的元素顺序，将该 collection 中的所有元素添加到此列表的尾部。
    */
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  //扩容检测
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
    *从指定的index位置开始，将指定 collection 中的所有元素插入到此列表中。
    */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index); //判断索引是否正确
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  //扩容检测
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData,index + numNew,numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
    * 用指定的元素替代此列表中指定位置上的元素。
    */
    public E set(int index, E e) {
        rangeCheck(index);  //判断索引是否正确
        checkForComodification();
        E oldValue = ArrayList.this.elementData(offset + index);
        ArrayList.this.elementData[offset + index] = e;
        return oldValue;
    }
```

### 2.4 ArrayList的删除操作  
ArrayList提供了四个方法进行元素的删除：
1. remove(int index)
2. remove(Object o)
3. removeRange(int fromIndex, int toIndex)
4. removeAll():是继承自AbstractCollection的方法，ArrayList本身并没有提供实现。

```java
    /**
    * 移除此列表中指定位置上的元素。
    */
    public E remove(int index) {
        rangeCheck(index);
        modCount++; 
        E oldValue = elementData(index);//需要删除的元素
        int numMoved = size - index - 1;
        if (numMoved > 0)//若需要移动，则向左移动numMoved位
            System.arraycopy(elementData, index+1,elementData, index,numMoved);
        elementData[--size] = null; //置空最后一个元素
        return oldValue;
    }

    /**
    * 移除此列表中首次出现的指定元素（如果存在）。
    */
    public boolean remove(Object o) {
        //因为ArrayList中允许存在null，所以需要进行null判断
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    //其中fastRemove()方法用于移除指定位置的元素。如下
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; 
    }

    /**
    * 移除列表中索引在 fromIndex（包括）和 toIndex（不包括）之间的所有元素。
    */
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,numMoved);
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }
```

### 2.5 ArrayList的查找操作
ArrayList提供了get(int index)用读取ArrayList中的元素。由于ArrayList是动态数组，所以我们完全可以根据下标来获取ArrayList中的元素，而且速度还比较快，故ArrayList长于随机访问。
```java
    public E get(int index) {
        rangeCheck(index); //检查下标是否合法
        return elementData(index);
    }
```
