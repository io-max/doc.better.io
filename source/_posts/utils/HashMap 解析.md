---
title: HashMap 解析
tags:
  - Java
  - Map
  - 源码分析
categories: HashMap
index_img: img/java-logo-480.png
abbrlink: 26563
---

## 本章简介

> 本章讲解如下：
>
> 1. HashMap 的常用方法的使用：put，set，remove
> 2. HashMap 是如何解决 hash 冲突？
> 3. HashMap 的时间复杂度/空间复杂度？
> 4. HashMap 底层源码分析
<!-- more -->


## HashMap


### 核心

HashMap的实例具有两个影响其性能的参数：`初始容量`和`负载因子`。容量是哈希表中存储桶的数量，初始容量只是创建哈希表时的容量。负载因子是散列表的容量自动增加之前允许其填充的完整程度的度量。当哈希表中的条目数超过负载因子和当前容量的乘积时，哈希表将被重新哈希（即，内部数据结构将被重建），因此哈希表的存储桶数约为两倍。



通常，默认负载因子（0.75）在时间和空间成本之间提供了一个很好的折衷方案。较高的值会减少空间开销，但会增加查找成本（在HashMap类的大多数操作中都得到体现，包括get和put）。





#### 常量

```java
// 默认容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 转变红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;

// 不转变红黑树的阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 红黑树最小的容量
static final int MIN_TREEIFY_CAPACITY = 64;
```



#### 变量

```java
/* ---------------- Fields -------------- */

// 数组，用于存放链表。第一次使用时初始化
transient Node<K,V>[] table;

transient Set<Map.Entry<K,V>> entrySet;

// 长度
transient int size;

// 记录 rehash 的计数
transient int modCount;

// 当前容量阈值
int threshold;

// 当前加载因子
final float loadFactor;
```



### 构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal initial capacity: " +
                                       initialCapacity);
  if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
  if (loadFactor <= 0 || Float.isNaN(loadFactor))
    throw new IllegalArgumentException("Illegal load factor: " +
                                       loadFactor);
  this.loadFactor = loadFactor;
  this.threshold = tableSizeFor(initialCapacity);
}
```



### 链表节点



#### 核心成员

```java
final int hash;  // hash 值
final K key;     // 存储的 Key
V value;   			 // 存储的 Value
Node<K,V> next;  // 后继节点，形成单向链表
```



#### 核心构造函数

```java
Node(int hash, K key, V value, Node<K,V> next) {
  this.hash = hash;
  this.key = key;
  this.value = value;
  this.next = next;
}
```



### 核心方法



HashMap 的核心方法这里只讲解 put，get，remove 三个方法。



#### put 方法

put 方法用于向 HashMap 中添加元素，支持



##### 源代码

```java
public V put(K key, V value) {
  return putVal(hash(key), key, value, false, true);
}
```

调用 `hash()` 方法计算 key 的 hash 值(此方法后面会讲解)。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
  Node<K,V>[] tab; // 当 hashMap 的数组
  Node<K,V> p;   // 数组中的节点
  int n, i;
  // 数组为空则调用 resize 方法对 table 进行初始化
  if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
  // 通过数组长度 & hash 值找到，此 hash 所在的节点
  if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
  // 节点不为空
  else {
    Node<K,V> e; 
    K k;
    // hash 碰撞
    if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
      e = p;
    // 如果是 TreeNode 则调用
    else if (p instanceof TreeNode)
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		// hash 未碰撞，向后遍历查找
    else {
      for (int binCount = 0; ; ++binCount) {
        // 后继节点为空，则直接创建 Node 节点
        if ((e = p.next) == null) {
          p.next = newNode(hash, key, value, null);
          // 如果达到树节点的阈值，则调用treeifyBin将节点（链表）改为树结构
          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
          break;
        }
        // 后继节点不为空且 key-value 都相同，直接结束循环
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
          break;
        // 不同则继续向后查找
        p = e;
      }
    }
   	// 节点不为空，说明存在映射的 key
    if (e != null) {
      V oldValue = e.value;
      if (!onlyIfAbsent || oldValue == null)
        e.value = value;
      afterNodeAccess(e);
      return oldValue;
    }
  }
  // 记录 rehash 的次数
  ++modCount;
  // 超过阈值则进行扩容
  if (++size > threshold)
    resize();
  afterNodeInsertion(evict);
  return null;
}
```



##### 执行流程

1. 如果 HashMap 未初始化，则调用 `resize `方法进行初始化。
2. 通过 `数组长度(table.length - 1)` **&** `hash 值`计算出 index 并判断链表头节点是否为空
   1. 链表头节点为空，说明可以新增节点，成功则执行步骤 4
   2. 链表头节点不为空，执行步骤 3
3. 当链表头节点不为空，有一下三种结果：
   1. 当前 hash 与链表头节点产生 hash 碰撞。
   2. 链表节点为 TreeNode，说明此时数据结构不在是链表而是树结构了
   3. 当前 hash 和链表头节点未产生 hash 碰撞 
      1. 从链表头节点向后遍历，有两种状况：
         1. next 节点与当前 hash 未产生碰撞，继续向后遍历，计数+1
         2. next 节点为空，则直接创建新节点并添加到链表中，当计数超过树阈值则当前链表结构会升级为树结构（`treeifyBin方法`）, 并结束循环。
   4. 产生碰撞后说明存在映射的 key，则替换旧值。
4. 记录 hash 次数，判断 size 是否超过阈值，超过则调用`resize`进行扩容



##### 树节点

当遍历链表时长度超过了 8 个，链表结构就会被淘汰升级会树结构，其数据结构就是 `TreeNode`。

后面会讲解 TreeNode



#### get 方法

get 方法用于根据指定的 key 从 HashMap 中获取 value

##### 源代码

```java
public V get(Object key) {
  Node<K,V> e;
  return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

查看 getNode 方法

```java
final Node<K,V> getNode(int hash, Object key) {
  Node<K,V>[] tab; 
  Node<K,V> first,
  Node<K,V> e; 
  int n; 
  K k;
  // table 不为空 且 节点不为空
  if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) 
  {
    // 头节点hash 与当前 hash 一致
    if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
      return first; // 直接返回
    // 后继节点不为空
    if ((e = first.next) != null) {
      // 节点为树形结构
      if (first instanceof TreeNode)
        return ((TreeNode<K,V>)first).getTreeNode(hash, key);
			// 开启循环， 遍历链表
      do {
        // 判断 hash 是否一致，一致则直接返回
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          return e;
      } while ((e = e.next) != null);
    }
  }
  return null;
}
```

##### 执行流程

1. 判断 table 是否为空，为空直接返回 bull，不为空执行步骤 2
2. 获取到链表头节点，并判断是否与当前 hash 一致，一致则直接返回，否则执行步骤 3
3. 判断后继节点是否为空，为空直接返回 null ，不为空则执行步骤 4
4. 链表头节点为树结构，直接转成 TreeNode 获取对象，反之执行步骤 5
5. 开启循环，不断获取后继节点，不断的判断后继节点的 hash 是否与当前 hash 一致，一致则返回，反之则继续向后遍历，如果遇到后继节点为 null，则结束循环返回 null。



#### remove 方法

用于通过 key 删除 HashMap 中的数据

##### 源代码

```java
public V remove(Object key) {
  Node<K,V> e;
  return (e = removeNode(hash(key), key, null, false, true)) == null ?
    null : e.value;
}
```

进入 removeNode 方法，查看具体的删除细节

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
  Node<K,V>[] tab; 
  Node<K,V> p; 
  int n;
  int index;
  // 数组不为空，且指定 hash 位置的链表不为空
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (p = tab[index = (n - 1) & hash]) != null) {
    Node<K,V> node = null, 
    Node<K,V> e; 
    K k;
    V v;
    // 命中 hash
    if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
      node = p; // 赋值
    // 未命中
    else if ((e = p.next) != null) {
      if (p instanceof TreeNode)
        // 从树形节点中获取
        node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
      else {
        do {  // 开启循环，直到尾节点为 null
          // 判断后继节点是否命中 hash
          if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
            node = e;
            break;
          }
          // 未命中继续向后
          p = e;
        } while ((e = e.next) != null);
      }
    }
    // node 不为空说明 hash 命中了
    // 是否匹配 value 值
    if (node != null && (!matchValue || (v = node.value) == value ||
                         (value != null && value.equals(v)))) {
      // 
      if (node instanceof TreeNode)
        ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
			// 命中了链表的头节点
      else if (node == p)
        tab[index] = node.next;
			// 非头节点
      else
        p.next = node.next;
			// 修改值
      ++modCount;
      --size;
      afterNodeRemoval(node);
      return node;
    }
  }
  return null;
}
```



##### 执行流程

1. 判断数组是否为空，且 hash 索引处的链表是否为空，为空直接返回 null，反之执行步骤 2
2. 判断hash是否命中链表的头节点，命中执行步骤 4，未命中执行步骤 3
3. 开启循环，向后遍历链表，判断 hash 是否命中，命中则结束循环执行步骤 4，反之循环结束执行步骤 4
4. 判断是否匹配 value
   1. 链表升级为树，则调用 TreeNode 删除节点
   2. 命中链表头节点。
   3. 为命中链表头节点，修改 next 指针。





#### hash 算法

hash 方法用于计算 key 的 hash 值。

```java
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

举例：此时假设默认长度是 16，则 index 的索引计算查看低 4 为

key.hashCode:    1101 0100 1010 1010 1011 1111 0010 0111

h>>>16:               0000 0000 0000 0000 1101 0100 1010 1010 

hash ^ h>>>16:   1101 0100 1010 1010 0110 1011 1000 1101

hash 计算后再与`table.length-1` 进行`或运算`

代码： `tab[i = (n - 1) & hash]`，n 为数组长度， hash 则是通过或运算计算出的 hash

table.length-1 : 0000 0000 0000 0000 0000 0000 0000 1111

hash :                1101 0100 1010 1010 0110 1011 1000 1101

i:                        0000 0000 0000 0000 0000 0000 0000 1101  （13）



### 扩容

上面知道了当大小超过了阈值就会进行扩容，调用 resize 方法

#### 源代码

```java
final Node<K,V>[] resize() {
  Node<K,V>[] oldTab = table;
  int oldCap = (oldTab == null) ? 0 : oldTab.length;
  int oldThr = threshold;
  int newCap, newThr = 0;
  
  // 旧容量大于0
  if (oldCap > 0) {
    if (oldCap >= MAXIMUM_CAPACITY) {
      threshold = Integer.MAX_VALUE;
      return oldTab;
    }
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
      newThr = oldThr << 1; // double threshold
  }
  // 加载因子大于 0
  else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
	// 加载默认配置
  else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
  }
  // 长度阈值为 0，则重新计算
  if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
  }
  threshold = newThr;
  // 创建新数组
  Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
  table = newTab;
  // 旧数组不为空
  if (oldTab != null) {
    // 遍历数组中所有的链表
    for (int j = 0; j < oldCap; ++j) {
      Node<K,V> e;
      // 链表节点不为空
      if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        // 后继节点为空
        if (e.next == null)
          newTab[e.hash & (newCap - 1)] = e;
        // 树形结构
        else if (e instanceof TreeNode)
          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				// 后继节点不为空
        else {
          Node<K,V> loHead = null;  // 低位头节点
          Node<K,V> loTail = null;  // 低位尾节点
          Node<K,V> hiHead = null;  // 高位头节点
          Node<K,V> hiTail = null;  // 高位尾节点
          Node<K,V> next;
          
          do {
            next = e.next;
            // 索引在低位
            if ((e.hash & oldCap) == 0) {
              if (loTail == null)
                loHead = e;      
              else
                loTail.next = e; 
              loTail = e;        // 第一次赋值，tail 和 head 都是 e
            }
            // 索引在高位
            else {
              if (hiTail == null)
                hiHead = e;
              else
                hiTail.next = e;
              hiTail = e;
            }
            // 继续下一个节点
          } while ((e = next) != null);
          
          // 存储低位
          if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
          }
          // 存储高位
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



#### 执行流程

1. 重新计算阈值和加载因子
2. 创建新数组
3. 遍历旧数组，获取到每个下标的链表，为空则跳过继续下次循环
   1. 链表的后继节点为空，直接插入数组
   2. 链表的后继节点不为空，开启循环遍历链表
      1. 声明高位头尾节点，低位头尾节点
      2. 通过节点的 hash 与旧数组长度与运算，判断结果是否为 0
         1. 为 0 则是低位，赋值低位的头尾节点
         2. 不为 0 则是高位，赋值高位的头尾节点
      3. 判断高低位的头尾节点是否为空，不为空直接插入数组
   3. 节点为红黑树类型，调用 TreeNode.split 方法

## 文章

[Hash 解析](https://www.hollischuang.com/archives/2091)




