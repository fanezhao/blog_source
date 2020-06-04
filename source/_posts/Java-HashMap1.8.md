---
title: 手斯HashMap
date: 2020-06-03 13:30:00
tags:
  - Java
---

## 基础概念

### 数组

数组是用于储存多个相同类型数据的集合。

- 优势：内存连续、有下标、查找快。
- 劣势：增删慢、大小固定不可变。扩容需新建大容量数组，并转移原数据数据，耗性能。

### 链表

链表是一种物理存储单元上非连续、非顺序的存储结构，数据元素的逻辑顺序是通过链表中的指针链接次序实现的。链表由一系列结点（链表中每一个元素称为结点）组成，结点可以在运行时动态生成。每个结点包括两个部分：一个是存储数据元素的数据域，另一个是存储下一个结点地址的指针域。 

- 优势：增删快、无限扩容。
- 劣势：查找慢。

数组和链表分别有其优势和劣势，那有没有一种数据结构有其两者的优势呢？有，散列表。

### 散列表

散列表（Hash table，也叫哈希表），是根据关键码值(Key value)而直接进行访问的数据结构。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做散列函数，存放记录的数组叫做散列表。

散列表特点：数组+链表。

### 什么是hash？

核心理论：Hash也称散列、哈希。基本原理就是把**任意长度**的输入，通过hash算法变成**固定长度**的输出。这个映射的规则就是对应的hash算法，而原始数据结构映射后的二进制串就是哈希值。

特点：

- 1、从hash值不可推导出原有数据。

- 2、输入数据的微小变化会得到完全不同的hash值，相同的数据会得到相同的hash值。

- 3、hash算法的执行效率要高效，长文本也要快速计算出hash值。

- 4、hash算法的冲突概率要小。

由于hash的原理是将输入空间的值映射成hash空间内，而hash值的空间远小于输入空间。根据**抽屉原理**，一定会存在不同的输入映射成相同输出的情况。

抽屉原理：桌上有10个苹果，要把这10个苹果放到9个抽屉里，无论怎样做，我们会发现至少有一个抽屉里面放了**不少于2个**苹果，这一现象就是我们说的抽屉原理。

## HashMap核心源码解析

### HashMap继承体系

{% qnimg Map继续体系.jpeg %}

### 核心属性

```java
// table数组默认大小
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// table数组最大长度
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表树化阈值
static final int TREEIFY_THRESHOLD = 8;

// 树降级成链表阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 树化另一个参数，当哈希表中的所有元素个数超过64时，才会树化
static final int MIN_TREEIFY_CAPACITY = 64;

// HashMap的底层存储结构其实就是Node数组+链表+红黑树
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;	// 哈希值
    final K key;
    V value;
    Node<K,V> next;	// 下个节点

	// ... ...
}

// 哈希表
transient Node<K,V>[] table;

// 当前哈希表中元素个数
transient int size;

// 当前哈希表结构修改次数
transient int modCount;

// 扩容阈值
// threshold = capacity * loadFactor
int threshold;

// 负载因子
final float loadFactor;

```

### 构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
    // 校验capacity
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 校验loadFactor
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 返回一个大于等于当前值cap的一个且是2的次方数的数值
    this.threshold = tableSizeFor(initialCapacity);
}

// cap = 10; n = 9
// 0b1001 | 0b0100 = 0b1101
// 0b1101 | 0b0011 = 0b1111
// 0b1111 | 0b0000 = 0b1111
// ...
// return 16;
static final int tableSizeFor(int cap) {
    // 减1是为了获取大于等于当前值的2的次方的值
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

构造方法流程总结：

- 1、校验数组容量和负载因子的合法性。
- 2、对数组容量进行移位处理，使其低4位为1111，获取一个大于目标容量的2的次方数。

### put()方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 作用：让key的hash值的高16位也参与路由运算，散列的更加均匀。
// 异或：相同则返回0，不同则返回1
// h = 0b 0010 0101 1010 1100 0011 1111 0010 1110
// 0b 0010 0101 1010 1100 0011 1111 0010 1110
// ^
// 0b 0000 0000 0000 0000 0010 0101 1010 1100
// => 0010 0101 1010 1100 0001 1010 1000 0010
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // tab：引用当前HashMap的散列表
    // p：表示当前散列表的元素
    // n：表示散列表数组的长度
    // i：表示路由寻址结果
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 延迟初始化逻辑。第一次调用putVal会初始化hashMap中的最耗费内存的散列表
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 通过寻址找到的桶位是null，直接将k-v封装成一个Node节点放到此处。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // e：桶位不为null的话，表示找到了一个与当前要插入k-v相同的key的节点
        // k：表示临时的一个k
        Node<K,V> e; K k;
        
        // 当前当前桶位中的元素的key与要插入元素的key一致，表示后续需要替换操作。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 如果当前桶位链表已经树化，则执行红黑树的相关操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 仍然是链表的情况。并且链表的关元素与我们要插入的key不一致
            for (int binCount = 0; ; ++binCount) {
				// 如果当迭代到链表的最后一个节点仍未找到与要插入k-v的key一致的节点，
                // 则将k-v封装成一个新的Node并加入到链表的末尾
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 新增节点之后，如果当前链表的长度大于等于链表树化的阈值，则触发树化操作
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 树化操作
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到一个与新增k-v的key一致的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 找到一个与新增k-v的key一致的节点，替换其value，返回老值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 散列表被修改的次数+1，替换不算
    ++modCount;
    // 如果散列表中的Node数量大于扩容阈值，触发扩容。
    if (++size > threshold)
        // 扩容操作
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

put()方法流程总结：

- 1、先对key进行路由运算，算法是：h ^ h >>> 16（当前key的hash值异或哈希值右移16位）。这个右移的目的主要是为了使key的hash值的高16位也参与运算，使散列更加均匀。

- 2、第二步判断当前hashMap的table是否已经被初始化了。如果没有，则对其进行初始化。延迟初始化的主要目的是为了防止初始化而不用会导致内存浪费，所以选择在第一次put操作的时候进行初始化。

- 3、对当前k-v进行插入操作，这里分为三种情况：

  - a、如果当前的桶位为 null 的话，就直接将 k-v 封装成一个 Node 放到此处即可。
  - b、如果当前桶位的链表已经树化，则执行红黑树的添加操作。
  - c、如果当前桶位的链表还未树化，则循环该链表，循环的过程中又分为两种情况：
    - aa、如果循环到链表的末尾仍未找到与当前插入元素的相同节点，则将 k-v 封装成一个新的 Node 加到链表的末尾，结束循环。
    - bb、如果找到与当前插入元素的相同节点，则将此节点赋值给临时变量e，结束循环。

  如果临时变量e不为 null，则说明找到了相同key的节点，然后会进行替换操作，返回旧值。

- 4、然后会对modCount自增，再对size自增。

- 5、如果 size 大于扩容阈值，则会触发扩容操作。

### resize()方法

```java
final Node<K,V>[] resize() {
    // oldTab：引用扩容前的哈希表
    Node<K,V>[] oldTab = table;
    // oldCap：扩容之前的哈希表的长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // oldThr：扩容之前的扩容阈值
    int oldThr = threshold;
    // newCap：扩容之后的长度
    // newThr：扩容之后，下次的扩容阈值
    int newCap, newThr = 0;
    
    // 如果hashMap散列表已经初始化过了，说明这是一次正常的扩容
    if (oldCap > 0) {
        // 扩容之前的table数组大小已经达到了最大阈值之后，则不再扩容，且设置扩容条件为 int 最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // oldCap 左移一位实现翻倍，并且赋值给newCap
        // newCap 小于数组最大限制且扩容之前的长度>=16
        // 那么，下次扩容的阈值是本次的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // oldCap == 0 说明hashMap中的散列表是null
    // 1、new HashMap(initCap, loadFactor)
    // 2、new HashMap(initCap)
    // 3、new HashMap(map) 并且这个map有数据
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // oldCap == 0 且 oldThr == 0
    // new HashMap()
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // newThr为0时，通过newCap和loadFactor计算出一个新的newThr
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    // 创建出一个更大的数据
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 如果老的hash表中有数据，则开始进行数据转移操作
    if (oldTab != null) {
        // 遍历老的table数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 如果当前当前桶位有数据，则分为三种情况
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果当前桶位只有一个数据节点，从未发生过碰撞，则将其再次路由到新的数组中即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                
                // 如果当前节点已经树化，则进行相应的操作
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 当前桶位已经链化
                    
                    // 低位链表：新数组的下标位置与当前数组下标位置一致
                    Node<K,V> loHead = null, loTail = null;
                    // 高位链表：新数组的下标位置为 当前数组下标 + 当前数组长度
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 低位链操作
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 高位链操作
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
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

**为什么要扩容？**为了解决哈希冲突导致地链化影响查询效率问题。

resize()流程总结：

- 1、首先计算出新数组的长度和下次的扩容阈值。

- 2、根据新的数据长度创建一个新 Node数组。
- 3、循环遍历老数组，这里分为三种情况：
  - a、如果当前桶位只有一个数据节点的情况下，直接对其重新路由放到新数组对应的位置即可。
  - b、如果当前桶位已经树化，则对其执行红黑树的相关操作。
  - c、如果当前桶位已经链化，这里也分为两种情况：
    - aa、如果当前节点的hash值按位与旧数组的长度等于0，则将其加到低位链表上。
    - bb、如果当前节点的hash值按位与旧数组的长度等于1，则将其加到高位链表上。
  - 将低位和高位链表放到新和数组对应的位置上。其中，低位链表的在新数组中的索引等于旧数组的索引；高位链表的在老数组中的索引等于老数组的索引 + 老数组的长度；
- 返回新的数组。

### get()方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    // tab：引用当前hashMap的散列表
    // first：桶位中的头元素
    // e：临时Node元素
    // n：table的长度
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 定位的元素即是桶位元素，直接返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶位元素不等于所要查找的元素，分两种情况
        if ((e = first.next) != null) {
            // 如果当前桶位已经树化，则执行红黑树的查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 如果当前桶位已经链化，循环遍历链表
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

get()方法流程总结：

- 1、先判断当前hashMap的数组是否已经初始化，如果没有，返回null。
- 2、如果table已经初始化，则分为三种情况：
  - a、如果定位到的桶的元素刚好等于要找的元素，则直接返回即可。
  - b、如果桶位已经树化，则执行红黑树的查找操作。
  - c、如果桶位已经链化，则循环遍历链表。

### remove()方法

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    // tab：引用当前hashMap的散列表
    // p：当前Node元素
    // n：table的长度
    // index：寻址结果
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        // node 查找到的结果
        Node<K,V> node = null, e; K k; V v;
        // 当前桶位元素等于要删除的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 如果当前桶位已经树化，则执行红黑树的查找
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 如果当前桶位已经链化，循环遍历链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 如果找到对应的node
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 如果树化，则进行红黑树的删除操作
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果桶位元素等于的要删除的元素的话，直接将桶位元素置为null
            else if (node == p)
                tab[index] = node.next;
            // 如果链化，则进行链表的删除操作
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

remove()流程总结：

- 1、先判断当前hashMap的数组是否已经初始化，如果没有，返回null。
- 2、如果已经初始化，则先进行查找操作，找到之后，再进行 node 的删除操作
- 3、查找操作分为三种情况：
  - a、如果定位到的桶的元素刚好等于要找的元素，则直接返回即可。
  - b、如果桶位已经树化，则执行红黑树的查找操作。
  - c、如果桶位已经链化，则循环遍历链表。
- 4、找到之后删除操作也分为3种情况：
  - a、如果桶位已经树化，则执行红黑树的删除操作。
  - b、如果定位到的桶的元素刚好等于要找的元素，则直接删除即可。
  - c、如果桶位已经链化，则链表删除操作。

以上代码基于JDK1.8，其中有关HashMap中红黑树的操作，可以参考另一篇文章[红黑树](http://zmoyi.com/2020/06/05/DataStructure-RBTree/)。

## HashMap在JDK1.7和JDK1.8中的区别

|              | JDK1.7                                                       | JDK1.8                                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 底层结构     | 数组+链表                                                    | 数组+链表+红黑树                                             |
| 新增节点方法 | 头插法                                                       | 尾插法                                                       |
| 扩容时机     | 节点插入之前扩容                                             | 节点插入之后扩容                                             |
| 寻址算法     | 扰动函数：<br />h = 0;<br />h ^= k.hashCode();<br />h ^= (h >>> 20) ^ (h >>> 12);<br />return h ^ (h >>> 7) ^ (h >>> 4);<br />+<br />h & (length-1) | 扰动函数：<br /> (h = key.hashCode()) ^ (h >>> 16)<br />+<br />(n - 1) & hash |