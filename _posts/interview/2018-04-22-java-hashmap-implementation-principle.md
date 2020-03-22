---
layout: post
title: 街题系列 - HashMap 实现原理。
category: 街题系列
tags: Java HashMap
---
* content
{:toc}
# HashMap
特点：
1. 基于Map接口实现
2. 运行null键/值,但是最多允许一条记录的键为null，允许多条记录的值为null
3. 非同步
4. 不保证有序，比如插入的顺序
5. 不保证顺序不随时间变化

## HashMap 的数据结构
Java中常见的两种结构是数组和模拟指针(引用)，几乎所有的数据结构都可以用这两种结构组合实现，HashMap也不例外，实际上HashMap是一个“链表散列”，结构如下
![](../../../../images//20170209185858523.jpeg)

从图中可以看出，HashMap底层还是数组，只是数组中每一项成为一个桶，即bucket，每一个桶只存一个元素，也就是一个Node对象,由于Node对象可以包含一个引用变量next用于指向另外一个Node,源码如下，因此可能出现，尽管桶里面只有一个Node对象，但是这个对象又指向另外一个Node对象，这样就形成了一条链，即Node链，而在JDK1.8 上，又添加了红黑树，当Node链的长度大于8的时候，就转换成一个红黑树，红黑树快速CRUD的特点提高了HashMap的效率
每一个Node链中对应的Key的hash（hashCode）返回值相同。

其中Capacity就是该数组的长度，这个长度是可以改变的，

### Node
是HashMap的一个内部类，实现了Map.Entry接口，它也维护这一个key-value映射关系，除了key-value,还有一个next引用，该引用指向当前table位置的链表，hash值，用来确定每一个Node链表在table的位置。

```java
static class Node<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        ...
}
```
## 两个重要参数
1. Capacity  容量 hash 表中桶的数量，默认从初始容量是16
2. Load foctor 负载因子 hash表在自动增加之前可以达到多满的一种程度。负载因子越大，表示散列表的装填程度越高。反之越小。   
负载因子存在的意思： 对于使用链表法的散列表来说，查找一个元素的平均时间是O(1+a),因此，负载因子越大，对空间的利用率越高，然而查找效率就越低，反之，负载因子越小，那么散列表数据过于稀疏，空间浪费严重，默认负载因子是0.75
当哈希表中条目数据超出了负载因子与容量的乘积的时候，则要对该hash表进行rehash操作，即重建内部数据，从而该哈希表具有两倍的桶数(buckets)

## 确定Hash桶数组索引位置。
不管增添，删除，或者查找，确定Hash桶数组索引的位置都是关键。HashMap 中使用hash算法求这个位置。
### 方法一
```java
static final int hash(Object key) {
       int h;
       // >>> 无符号右移,高位补0
       //高16bit不变，低16bit和高16bit做了一个异或。
       // h = key.hashCode() 为第一步 取hashCode值
      // h ^ (h >>> 16)  为第二步 高位参与运算
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
```
这儿方法在JDK1.8 有，主要包括三步
1. 如果key 为null，那么hash 值就是0，这个就是为啥hashmap能存null键的原因，把key为null的情况单独处理成0了
2. key不为null，取得hashcode 值，即 h = key.hashCode()
3. 高16bit不变，低16bit和高16bit做了一个异或(相同为0，不同为1 ) ，h^(h >>> 16),这么做主要是从速度，功效，质量来考虑的，这样做能保证在table数组的length比较小的时候，也能考虑到高低位都参与到hash 计算中， 同时不会有太大的开销。


### 方法二
```java
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算 等价于h%(length-1)
}
```  
这个方法JDK1.7 上有，JDK1.8 上没有，但是原理一样，

Hash 算法的本质分三步
1. 取key的hashCode
2. 高位运算
3. 取模运算

HashMap 采用方法二计算该对象应该保存到table数组中哪一个索引处。

举例说明，
1. 假如key的hashcode值 h 为1111 1111 1111 1111 1111 0000 1110 1010
2. 进行无符号右移16位，高位补0 即h>>>16 为0000 0000 0000 0000 1111 1111 1111 1111
3. 步骤1 和步骤2 的值进行异或(相同为0，不同为1 )操作，结果为hash 即hash=h^(h>>>16) ，即  

```java
  1111 1111 1111 1111 1111 0000 1110 1010   
^ 0000 0000 0000 0000 1111 1111 1111 1111
-------------------------------------------
  1111 1111 1111 1111 0000 1111 0001 0101
```
所以最后hash的值就是 1111 1111 1111 1111 0000 1111 0001 0101

4. 取模运算，用n-1把步骤三的结果取模 ，因为我们知道，n是2的幂次方结果，默认是16，所以n-1 肯定是15，即 1111

```java
  0000 0000 0000 0000 0000 0000 0000 1111
& 1111 1111 1111 1111 0000 1111 0001 0101
------------------------------------------
  0000 0000 0000 0000 0000 0000 0000 0101
```
即值为0101，也就是5

## HashMap 的put方法，
这里说的是JDK1.8的put方法流程
![](../../../../images/hashmap_put_method.png)
1. 判断键值对数组table[i]是否为null或者为空，是的话就执行resize进行扩容
2. 根据键key计算hash值得到插入数组的索引。如果table[i]为null，直接新建节点插入，转向 6.否则转向3
3. 判断table[i]的首个元素是否和key一样，如果相同（hashcode 和equals()都相等）则直接覆盖，否则转向4
4. 判断table[i]是否为TreeNode,即红黑树，如果是则在红黑树上执行插入操作，否则转向5
5. 遍历table[i],判断链表长度是否大于8，大于8则把链表转换成红黑树，然后在红黑树上执行插入操作。否则在链表中进行插入操作，遍历中发现key已经存在，则直接覆盖value即可
6. 插入成功后，判断实际存在的键值对是否大于最大容量的threshod，如果超过了，则进行扩容

源码如下

```java  
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i;
        //tab 为null的时候，创建
        if ((tab = table) == null || (n = tab.length) == 0) {
            n = (tab = resize()).length;
        }
        //(p = tab[i = (n - 1) & hash])
        i = (n - 1) & hash; //计算index
        p = tab[i];//得到tab中下标为 index 的值，
        if (p == null) {//说明该hash没有碰撞，直接创建一个Node,放到bucket中
            tab[i] = newNode(hash, key, value, null);
        } else {//说明碰撞上了
            Node<K, V> e;
            K k;
            k = p.key;//得到p对应的key
            if (p.hash == hash && (k == key || (key != null && key.equals(k)))) {//hash值相同，并且key相等，说明是同一个
                e = p;
            } else if (p instanceof TreeNode) {//该链为树
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
            } else {//该链为链表
                for (int binCount = 0; ; ++binCount) {
                    e = p.next;
                    if (e == null) {//p是处于链表尾部
                        p.next = newNode(hash, key, value, null);//那么就创建一个新的，然后连接到p后面
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  说明大于链表的阙值，应该转为红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    //key已经存在，则直接覆盖value
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        break;
                    }
                    p = e;
                }
            }
            //写入
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null) e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //超过load factor* current capacity ,进行扩容
        if (++size > threshold) {
            resize();
        }
        afterNodeInsertion(evict);
        return null;
    }
```

## 扩容机制
扩容（resize） 重新计算容器。java数组无法自动扩容，方法就是使用新数组代替已有的容量小的数组。
### JDK1.7 源码解析
```java
void resize(int newCapacity) {//穿入新的容量
    HashMapEntry[] oldTable = table;//引用扩展前的数组
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {//扩容前如果数组大小已经达到最大2^30了
        threshold = Integer.MAX_VALUE;//修改阙值为int的最大值，这样以后就不会在扩容了
        return;
    }

    HashMapEntry[] newTable = new HashMapEntry[newCapacity];//初始化一个新的数组
    transfer(newTable);//将数据转移到新的数组中
    table = newTable;//HashMap中的table属性指向新的数组
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);//修改阙值
}
```
这里面主要做五步
1. 扩容也是有个限度的，并不是无限扩容，当扩容数字数组大小达到了2^30，就不在扩容了，修改阙值为最大值，这样就不会在扩容了，只能随你碰撞了，
2. 创建一个新的数组，数组程度就是扩容后的长度
3. 把原来的数组中的元素转移到新的数组中
4. HashMap中的table属性指向新的数组
5. 修改阙值

这里面最主要的就是转移数组，也就是第三步，接下来咱们看看transfer()方法
```java
void transfer(HashMapEntry[] newTable) {
    int newCapacity = newTable.length;
    for (HashMapEntry<K,V> e : table) {//遍历旧的数组，取得数组中 的每一个元素
        while(null != e) {
            HashMapEntry<K,V> next = e.next;
            int i = indexFor(e.hash, newCapacity);//重新计算每个元素在数组中的位置。
            e.next = newTable[i]; //标记【i】
            newTable[i] = e;//将元素放到数组上
            e = next;//访问下一个Entry 链的元素
        }
    }
}
```
这里面做的事情其实也挺简单，想明白最后三行代码就可以了，
1. 循环遍历之前的旧的数组，
2. 取得每个元素，既然是转移，就一个都不能放过，因为这个元素可能有next 元素，所以需要再使用循环遍历每个桶内的Entry 链
3. 先保存下来Entry 的next元素
4.   e.next = newTable[i]; 重点解释这句话，其实这话也很好理解，我们先根据Entry的hashcode和容量得到该Entry 元素在新的数组中的位置i,这就是之前使用的indexFor()方法计算得到i
5. 因为这个i 所在的位置可能已经存在一个元素了，也就是所谓的发生了碰撞，所以新过来的元素就放到了链头，有点像是一个堆结构，先进后出，这个是先来的往后排，后来的往前排，所以不管新的数组中第i 个元素是否已经有Entry 元素，都要放到后来这个Entry的next位置，如果旧的链表迁移到新的链表，位置相同，则链表的位置会倒置过来，就是这个意思
6. 然后i 这个位置放入新的Entry 元素
7. 最后把e指向之前保存下来的next元素，直到e 为null，为止

接下来看JDK1.8 的扩容源码
JDK 1.8的优化
使用2次幂进行扩容的，所以元素的位置，要么在原来的位置，要么在原来的位置再移动2次幂的位置。  
在扩容的HashMap的时候，不需要像JDK1.7 那样重新结算hash,只需要看看原来的hash新增的那位bit值是1还是0就好，如果是0，索引不变，如果为1，索引变成原索引+oldCap，并且扩容后，JDK1.8不会倒置链表

```java
final Node<K, V>[] resize() {
      Node<K, V>[] oldTab = table;
      int oldCap = (oldTab == null) ? 0 : oldTab.length;
      int oldThr = threshold;
      int newCap, newThr = 0;
      if (oldCap > 0) {//说明不是初始化，
          //超过最大值就不在扩容，只好随你去碰撞吧
          if (oldCap >= MAXIMUM_CAPACITY) {
              threshold = Integer.MAX_VALUE;
              return oldTab;
              //没有超过最大值，就扩充到原来的二倍
          } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY) {
              newThr = oldThr << 1; // double threshold 新的阙值就是旧的阙值左移一位
          }
      } else if (oldThr > 0) {// initial capacity was placed in threshold{
          newCap = oldThr;//初始容量被放入阈值
      } else {               // zero initial threshold signifies using defaults 零初始阈值表示使用默认值
          newCap = DEFAULT_INITIAL_CAPACITY;
          newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
      }

      //计算新的resize上限
      if (newThr == 0) {
          float ft = (float) newCap * loadFactor;
          newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ? (int) ft : Integer.MAX_VALUE);
      }
      threshold = newThr;
      @SuppressWarnings({"rawtypes", "unchecked"})
      Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
      table = newTab;
      if (oldTab != null) {
          //把整个bucket都移动到新的bucket中
          for (int j = 0; j < oldCap; ++j) {
              Node<K, V> e = oldTab[j];
              if (e != null) {//说明该位置存有元素
                  oldTab[j] = null;//把原来的位置清空
                  if (e.next == null) {//说明该位置只有一个元素，
                      newTab[e.hash & (newCap - 1)] = e;//计算新的位置并把该元素放入新的位置
                  } else if (e instanceof TreeNode) {//说明该元素是一个红黑树
                      ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                  } else { // preserve order  维持秩序
                      Node<K, V> loHead = null, loTail = null;
                      Node<K, V> hiHead = null, hiTail = null;
                      Node<K, V> next;
                      do {
                          next = e.next;//保存该元素的next
                          //原索引
                          int newBit = e.hash & oldCap;//原来的hash值与旧的容量进行与操作，得到的就是bit是1还是0就好了
                          if (newBit == 0) {// 说明插入的新的坐标位置和原来的一直
                              if (loTail == null) {// 低位的末尾为null
                                  loHead = e; //低位的头就是该元素
                              } else {//低位的末尾不为null，说明这个位置已经有元素了，也就是发生了碰撞
                                  loTail.next = e;//那么这个元素就排在后面，而不是插在已有的位置前面
                              }
                              loTail = e;//该低位尾部指向该元素
                          } else {//原索引加上oldcap 说明插入的新的坐标位置和原来的不一样
                              if (hiTail == null) {//高位的末尾为null
                                  hiHead = e;//那么高位的头元素就是该元素
                              } else {//
                                  hiTail.next = e;
                              }
                              hiTail = e;//高位的尾部指向该元素
                          }
                      } while ((e = next) != null);

                      //原索引放入到buckets中
                      if (loTail != null) {
                          loTail.next = null;//低位的末尾设置为null
                          newTab[j] = loHead;//高位的新坐标就指向头低位的头元素
                      }
                      //原索引加上oldcap放入buckets中
                      if (hiTail != null) {
                          hiTail.next = null;
                          newTab[j + oldCap] = hiHead;//高位的新坐标加上就的数组长度指向新的头元素
                      }
                  }
              }
          }
      }
      return newTab;
  }
```
代码中都带有注释，就不过多解释了，相信能看懂的。

## HashMap 与HashTable的区别
1. HashMap 非线程安全，HashTable 线程安全
2. HashMap 的键值都允许为null,而HashTable则不行
3. 因为线程问题，哈希效率问题，HashMap 效率比HashTable的高
4. HashMap默认初始化大小是16，HashTable 的为11，HashMap 扩容是乘以2，使用位运算取得hash,效率高于取模运算，而后者扩容是乘以2加一，都是素数和计数，这样取模哈希结果更均匀。

Java中另外一个线程安全，功能与HashMap类似的是ConcurrentHashMap,
它和HashMap 的区别：
ConcurrentHashMap 也是不允许键值为 null，但是他线程安全，并且对整个桶组进行了分割，然后再每一段上都用了lock锁进行保护，相对于HashTable的syn关键字锁的粒度更精细一些，并发性更好一些，



---
搬运地址：  
[Java容器（四）：HashMap（Java 7）的实现原理](https://blog.csdn.net/jeffleo/article/details/54946424)   
[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)   
[HashMap 里的“bucket”、“负载因子” 介绍
](https://blog.csdn.net/wenyiqingnianiii/article/details/52204136)   
[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)   
[ Java集合——HashMap、HashTable以及ConCurrentHashMap异同比较](https://blog.csdn.net/seu_calvin/article/details/52653711)
