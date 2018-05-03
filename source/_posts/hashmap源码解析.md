---
title: hashmap源码解析
date: 2018-04-26 17:48:16
---
##源码第一篇(hashmap)简单理解篇
本篇讲解hashmap的put 方法的原理和一些源码的解析

下面先简单的介绍一下hashmap
(hashmap包位置为import java.util.HashMap)

hashmap的数据结构是数组和链表的结合
为什么会是这个结构，后面会讲到

hashmap是线程不安全的
非线程安全是指多线程操作同一个对象可能会出现问题
也就是说，我新建一个对象，用100个线程同时对这个对象进行操作，有可能是无法达到你预期的结果的。
至于原因大家应该都知道，是没有使用同步锁，所以才会导致这个问题的产生

所以这里举一反三，带有同步锁的是线程安全的，比如ArrayList是非线程安全的，Vector是线程安全的；HashMap是非线程安全的，HashTable是线程安全的；StringBuilder是非线程安全的，StringBuffer是线程安全的，那么线程安全的会因为同步锁的关系，在性能上会有所降低

<!--more-->
进入正题
当我们调用put方法放入key的时候，hashmap会调用hashcode方法获取哈希码，通过哈希码进行计算快速找到某个存放位置，这个位置可以被称之为bucketIndex，这个index的计算公式为
```
  static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```
这里涉及到取模等一系列复杂的运算，不是本篇的重点，大家对这个原理感兴趣的可以去研究下，为什么这里要使用lenth-1为什么会进行与计算等等

现在暂时你只需要知道这里可以得到value的放置位置bucketIndex
拿到bucketIndex后，就会把值存放在这个位置。

接下来简单介绍一下俩个方法
hashCode
equals

hashcode用来获取哈希码，equals用来比较字符串中所包含的内容是否相同
如果hashCode不同，equals一定为false，如果hashCode相同，equals不一定为true

那么这里就会出现一个问题，hashmap的键值对是唯一的，但是却又可能存在hashcode相同的情况，这个时候bucketIndex也会相同，那么hashmap是如何解决的呢？
当这个冲突发生的时候，也可以称这个冲突为"碰撞"，当碰撞发生后，hashmap会拿出bucketIndex上所有的值，通过hashCode和equals最终判断出新进来的key是否已经存在，如果存在就覆盖掉旧的值，如果不存在，就存入新的值。
上面我说hashmap会拿出bucketIndex上所有的值，从这句话可以看出，其实在某一个bucketIndex位置上，可能会存有多个value，（当然这个现象是比较少见的），所以hashmap在出现碰撞的时候，是通过一个链表来维护这个关系。

这也是hashmap数据结构的构成原理

那么知道大概的流程之后看一下相对于的源码会更加清晰，

下面是hashmap put的源码
```
    public V put(K key, V value) {
  
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);
        int i = indexFor(hash, table.length);
        for (HashMapEntry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```
  第一行，如果table为空，那么就实例化一个新表inflateTable(threshold);
  hashmap中有一个table用来存取value值
在源码顶部的定义处可以看到HashMapEntry<K,V>[] table = (HashMapEntry<K,V>[]) EMPTY_TABLE;

进入inflateTable这个方法看看
```
   private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        int capacity = roundUpToPowerOf2(toSize);

        // Android-changed: Replace usage of Math.min() here because this method is
        // called from the <clinit> of runtime, at which point the native libraries
        // needed by Float.* might not be loaded.
        float thresholdFloat = capacity * loadFactor;
        if (thresholdFloat > MAXIMUM_CAPACITY + 1) {
            thresholdFloat = MAXIMUM_CAPACITY + 1;
        }

        threshold = (int) thresholdFloat;
        table = new HashMapEntry[capacity];
    }
```
从最后俩句可以看出如果table 为空就会new出这个对象

接下来继续往下看
```
    if (key == null)
    return putForNullKey(value);
```
hashmap支持存取null值，这里对key为null的进行单独处理

进入putForNullKey方法
```
   private V putForNullKey(V value) {
        for (HashMapEntry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
```
对table表进行循环判断，判断是否有键为null的存在，如果存在，用新的value值，取代旧值。

如果不存在就使用addEntry()进行添加;

进入这个方法看下。
```
  void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? sun.misc.Hashing.singleWordWangJenkinsHash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

    /**
     * Like addEntry except that this version is used when creating entries
     * as part of Map construction or "pseudo-construction" (cloning,
     * deserialization).  This version needn't worry about resizing the table.
     *
     * Subclass overrides this to alter the behavior of HashMap(Map),
     * clone, and readObject.
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMapEntry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new HashMapEntry<>(hash, key, value, e);
        size++;
    }
```
简单的可以理解为找到bucketIndex值，创建Entry，看下createEntry方法里面
```
   HashMapEntry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new HashMapEntry<>(hash, key, value, e);
        size++;
```
很简单，是根据bucketIndex新建hashmapentry
同时总大小size++

到这里put的整个流程差不多就说完了，如果能完全了解这个流程，你在去看get方法，我相信会很清晰了





