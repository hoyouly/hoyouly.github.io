---
layout: post
title: 街题系列之---HashMap实现原理。
category: 扫盲系列
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
![](http://p5sfwb51p.bkt.clouddn.com/20170209185858523.jpeg)

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
![](http://p5sfwb51p.bkt.clouddn.com/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
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




---
搬运地址：  
[Java容器（四）：HashMap（Java 7）的实现原理](https://blog.csdn.net/jeffleo/article/details/54946424)   
[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)   
[HashMap 里的“bucket”、“负载因子” 介绍
](https://blog.csdn.net/wenyiqingnianiii/article/details/52204136)   
[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)
