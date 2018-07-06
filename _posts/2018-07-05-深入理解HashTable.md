---
layout:     post
title:      深入理解HashTable
subtitle:   源码分析
date:       2018-07-05
author:     PyP
header-img: img/post-bg-os-metro.jpg
catalog: 	 true
tags:
    
---

# 简介  
与jdk1.8之前版本的hashMap一样，HashTable是一个散列表，它存储的内容是键值对映射，HashTable继承于Dictionary，实现了Map，cloneable，Serializable接口。HashTable的函数都是同步的，这意味着它时线程安全的。它的key和value都不能为null，而且HashTable的映射都是无序的。  
hashTable的实例有两个参数影响其性能：**初始容量** 和 **加载因子**。容量是哈希表中桶的数量，初始容量是哈希表创建时的容量，默认是11，注意，哈希表的状态为 open：在发生“哈希冲突”的情况下，单个桶会存储多个条目，这些条目必须按顺序搜索。加载因子 是对哈希表在其容量自动增加之前可以达到多满的一个尺度。初始容量和加载因子这两个参数只是对该实现的提示。关于何时以及是否调用 rehash 方法的具体细节则依赖于该实现。
通常，默认加载因子是 0.75, 这是在时间和空间成本上寻求一种折衷。加载因子过高虽然减少了空间开销，但同时也增加了查找某个条目的时间（在大多数 Hashtable 操作中，包括 get 和 put 操作，都反映了这一点）。一次扩容增量为2倍+1.
***
# 结构分析

### 构造函数
>>	// 默认构造函数。
	public Hashtable()  
// 指定“容量大小”的构造函数  
	public Hashtable(int initialCapacity)   
 // 指定“容量大小”和“加载因子”的构造函数  
 public Hashtable(int initialCapacity, float loadFactor)   
 // 包含“子Map”的构造函数  
public Hashtable(Map<? extends K, ? extends V> t)

### API
>>synchronized void                clear()  
synchronized Object              clone()  
             boolean             contains(Object value)  
synchronized boolean             containsKey(Object key)  
synchronized boolean             containsValue(Object value)  
synchronized Enumeration<V>      elements()  
synchronized Set<Entry<K, V>>    entrySet()  
synchronized boolean             equals(Object object)  
synchronized V                   get(Object key)  
synchronized int                 hashCode()  
synchronized boolean             isEmpty()  
synchronized Set<K>              keySet()  
synchronized Enumeration<K>      keys()  
synchronized V                   put(K key, V value)  
synchronized void                putAll(Map<? extends K, ? extends V> map)  
synchronized V                   remove(Object key)  
synchronized int                 size()  
synchronized String              toString()  
synchronized Collection<V>       values()

***

# HashTable数据结构
### HashTable 的继承关系
>>java.lang.Object  
   ↳     java.util.Dictionary<K, V>  
         ↳     java.util.Hashtable<K, V>  
public class Hashtable<K,V> extends Dictionary<K,V>  
   &nbsp;&nbsp; implements Map<K,V>, Cloneable, java.io.Serializable { }
   
### HashTable 和 Map 关系如下图：
![关系图](https://ws1.sinaimg.cn/large/006tNc79ly1fsytz2janij30cv0db3yw.jpg)
从图中可以看出来：  
1. HashTable继承于Dictinary类，实现了Map接口，Map是“key-value键值对”接口，Dictionary是声明了操作“键值对”函数接口的抽象类。  
2. HashTable是通过“拉链法”实现的哈希表。它包括几个重要的成员变量：table，count，threshold，loadfactor， modCount。  
**table**是一个Entry[]数组类型，而Entry实际上就是一个单项链表。哈希表的“key-value”都是存储在Entry数组中的。  
**count**是hashtable的大小，保存了HashTable的键值对的数量。   
**threshold**是HashTable的阀值，用于判断是否需要调整HashTable的容量。threshold的值=“容量*加载因子”   
**loadfactory**加载因子
**modCount**用来记录修改次数，实现fail-fast机制的。 

### 源码解析
* 数据结点entry的数据结构

		private static class Entry<K,V> implements Map.Entry<K,V> { 
	    // 哈希值  
	    int hash;  
	    K key;  
	    V value;  
	    // 指向的下一个Entry，即链表的下一个节点  
	    Entry<K,V> next;  
	    // 构造函数  
	    protected Entry(int hash, K key, V value, Entry<K,V> next) {  
	        this.hash = hash;  
	        this.key = key;  
	        this.value = value;  
	        this.next = next;  
	    }  
	    protected Object clone() {  
	        return new Entry<K,V>(hash, key, value,  
	              (next==null ? null : (Entry<K,V>) next.clone()));  
	    }  
	    public K getKey() {  
	        return key;  
	    }  
	    public V getValue() {  
	        return value;  
	    }  
	    // 设置value。若value是null，则抛出异常。  
	    public V setValue(V value) {  
	        if (value == null)  
	            throw new NullPointerException();  
	        V oldValue = this.value;  
	        this.value = value;  
	        return oldValue;  
	    }  
	    // 覆盖equals()方法，判断两个Entry是否相等。  
	    // 若两个Entry的key和value都相等，则认为它们相等。  
	    public boolean equals(Object o) {  
	        if (!(o instanceof Map.Entry))  
	            return false;  
	        Map.Entry e = (Map.Entry)o;  
	        return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&  
	           (value==null ? e.getValue()==null : value.equals(e.getValue()));    
	    }  
	    public int hashCode() {  
	        return hash ^ (value==null ? 0 : value.hashCode());  
	    }  
	    public String toString() {  
	        return key.toString()+"="+value.toString();  
	    }  
		}

我们可以看出来Entry本身就是一个单向链表。这也就是我们说HashTable是通过拉链法解决冲突的。Entry实现了Map.Entry接口，即实现了getKey(),getvalue(),setValue(V value),equals(Object o),hashCode()这些函数。   

### hashTable主要对外接口
* clear()  
作用就是清空HashTable。它是将HashTable的table数组的值全部设为null。

		public synchronized void clear() {
	    Entry tab[] = table;
	    modCount++;
	    for (int index = tab.length; --index >= 0; )
	        tab[index] = null;
	    count = 0;
		}

* contains() 和 containsValue()  
判断HashTable是否包含该value
		
		public boolean containsValue(Object value) {
	    return contains(value);
		}
		
		public synchronized boolean contains(Object value) {
		    // Hashtable中“键值对”的value不能是null，
		    // 若是null的话，抛出异常!
		    if (value == null) {
		        throw new NullPointerException();
		    }
	
	    // 从后向前遍历table数组中的元素(Entry)
	    // 对于每个Entry(单向链表)，逐个遍历，判断节点的值是否等于value
	    Entry tab[] = table;
	    for (int i = tab.length ; i-- > 0 ;) {
	        for (Entry<K,V> e = tab[i] ; e != null ; e = e.next) {
	            if (e.value.equals(value)) {
	                return true;
	            }
	        }
	    }
	    return false;
		}

* containsKey()  
判断是否包含该key
	
		public synchronized boolean containsKey(Object key) {
	    Entry tab[] = table;
	    int hash = key.hashCode();
	    // 计算索引值，
	    // % tab.length 的目的是防止数据越界
	    int index = (hash & 0x7FFFFFFF) % tab.length;
	    // 找到“key对应的Entry(链表)”，然后在链表中找出“哈希值”和“键值”与key都相等的元素
	    for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
	        if ((e.hash == hash) && e.key.equals(key)) {
	            return true;
	        }
	    }
	    return false;
		}
（Ps：HashTable与HashMap的索引计算方式不同，HashTable采用 index = （hash& 0x7fffffff）% tab.length 的方式计算，其中0x7fffffff是Integer.MAX_VALUE,也就是2的32次幂-1，（hash & 0x7fffffff）只对符号位有效，应该是为了过滤负数，后面的取模是为了将index的值限制在数组的长度之内）
* elements()
	返回所有的value的枚举对象
	
			public synchronized Enumeration<V> elements() {
		    return this.<V>getEnumeration(VALUES);
		}
		
		// 获取Hashtable的枚举类对象

		private <T> Enumeration<T> getEnumeration(int type) {
		    if (count == 0) {
		        return (Enumeration<T>)emptyEnumerator;
		    } else {
		        return new Enumerator<T>(type, false);
		    }
		}
若HashTable的实际大小为0，则返回“空枚举类”对象emptyEnumerator;否则返回正常的Enumerator的对象。（enumerator实现了迭代器和枚举两个接口）		

* get()  
通过key值获取value，没有返回null

		public synchronized V get(Object key) {
		    Entry tab[] = table;
		    int hash = key.hashCode();
		    // 计算索引值，
		    int index = (hash & 0x7FFFFFFF) % tab.length;
		    // 找到“key对应的Entry(链表)”，然后在链表中找出“哈希值”和“键值”与key都相等的元素
		    for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
		        if ((e.hash == hash) && e.key.equals(key)) {
		            return e.value;
		        }
		    }
		    return null;
		}

* put()
put方法是对外提供的接口，让HashTable对象可以通过put()将“key-value”添加到HashTable中

		// put是synchronized方法
		public synchronized V put(K key, V value) {
		    // 首先就是确保value不能为空
		    if (value == null) {
		        throw new NullPointerException();
		    }
	
	    // Makes sure the key is not already in the hashtable.
	    Entry<?,?> tab[] = table;
	    int hash = key.hashCode();
	    // 计算数组的index
	    int index = (hash & 0x7FFFFFFF) % tab.length;
	    @SuppressWarnings("unchecked")
	    Entry<K,V> entry = (Entry<K,V>)tab[index];
	    // 如果index处已经有值，并且通过比较hash和equals方法之后，如果有相同key的替换，返回旧值
	    for(; entry != null ; entry = entry.next) {
	        if ((entry.hash == hash) && entry.key.equals(key)) {
	            V old = entry.value;
	            entry.value = value;
	            return old;
	        }
	    }
	    // 添加数组
	    addEntry(hash, key, value, index);
	    return null;
		}

* addEntry()

		private void addEntry(int hash, K key, V value, int index) {
		    modCount++;
	
	    Entry<?,?> tab[] = table;
	    // 如果容量大于了阈值，扩容
	    if (count >= threshold) {
	        // Rehash the table if the threshold is exceeded
	        rehash();
	
	        tab = table;
	        hash = key.hashCode();
	        index = (hash & 0x7FFFFFFF) % tab.length;
	    }
	
	    // Creates the new entry.
	    @SuppressWarnings("unchecked")
	    Entry<K,V> e = (Entry<K,V>) tab[index];
	    // 在数组索引index位置保存
	    tab[index] = new Entry<>(hash, key, value, e);
	    count++;
		}

>> 通过<font color = "red">tab[index] = new Entry<>(hash, key, value, e);</font>这一行代码，并且根据Entry的构造方法，我们可以知道，Hashtable是在链表的头部添加元素的，而HashMap是尾部添加的，这点可以注意下。


* putAll()
将一个Map集合中的元素逐一添加到HashTable中

		public synchronized void putAll(Map<? extends K, ? extends V> t) {
	     for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
	         put(e.getKey(), e.getValue());
		}
* remove()
删除HashTable中键为key的元素

		public synchronized V remove(Object key) {
		    Entry tab[] = table;
		    int hash = key.hashCode();
		    int index = (hash & 0x7FFFFFFF) % tab.length;
		    // 找到“key对应的Entry(链表)”
		    // 然后在链表中找出要删除的节点，并删除该节点。
		    for (Entry<K,V> e = tab[index], prev = null ; e != null ; prev = e, e = e.next) {
		        if ((e.hash == hash) && e.key.equals(key)) {
		            modCount++;
		            if (prev != null) {
		                prev.next = e.next;
		            } else {
		                tab[index] = e.next;
		            }
		            count--;
		            V oldValue = e.value;
		            e.value = null;
		            return oldValue;
		        }
		    }
		    return null;
		}
* rehash()

	    protected void rehash() {
	        int oldCapacity = table.length;
	        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
   		 }
rehash方法简单来说就是将旧数组放到一边，创建一个扩容后的新数组，遍历旧数组的长度，将每个数据的链表链遍历并重新分配位置，如果存在哈希冲突就存到该index位置的链表头部。
（PS：这里的MAX_ARRAY_SIZE = Integer.MAX_VALUE-8,为为什么是这么一个值呢？官方api给出解释是：一些VM在数组中保留一些headerword，分配较大数组的尝试可能导致OutOfMeMyLogyError。实际上是可以分配到Integer.MAX_VALUE这么大的。）

***

# HashTable的遍历方式
### 遍历HashTable的键值对
根据entrySet()获取HashTable的“键值对”的set集合，根据Iterator迭代器遍历set集合

	// 假设table是Hashtable对象
	// table中的key是String类型，value是Integer类型
	Integer integ = null;
	Iterator iter = table.entrySet().iterator();
	while(iter.hasNext()) {
	    Map.Entry entry = (Map.Entry)iter.next();
	    // 获取key
	    key = (String)entry.getKey();
	        // 获取value
	    integ = (Integer)entry.getValue();
	}
	
### 通过Iterator遍历HashTable的键
第一步：根据keySet()获取Hashtable的“键”的Set集合。  
第二步：通过Iterator迭代器遍历“第一步”得到的集合。

	// 假设table是Hashtable对象
	// table中的key是String类型，value是Integer类型
	String key = null;
	Integer integ = null;
	Iterator iter = table.keySet().iterator();
	while (iter.hasNext()) {
	        // 获取key
	    key = (String)iter.next();
	        // 根据key，获取value
	    integ = (Integer)table.get(key);
	}
	
### 通过Iterator遍历Hashtable的值
第一步：根据value()获取Hashtable的“值”的集合。  
第二步：通过Iterator迭代器遍历“第一步”得到的集合。

	// 假设table是Hashtable对象
	// table中的key是String类型，value是Integer类型
	Integer value = null;
	Collection c = table.values();
	Iterator iter= c.iterator();
	while (iter.hasNext()) {
	    value = (Integer)iter.next();
	}
	
### 通过Enumeration遍历Hashtable的键
第一步：根据keys()获取Hashtable的集合。  
第二步：通过Enumeration遍历“第一步”得到的集合。

	Enumeration enu = table.keys();
	while(enu.hasMoreElements()) {
	    System.out.println(enu.nextElement());
	} 

### 通过Enumeration遍历Hashtable的值
第一步：根据elements()获取Hashtable的集合。  
第二步：通过Enumeration遍历“第一步”得到的集合

	Enumeration enu = table.elements();	
	while(enu.hasMoreElements()) {
	    System.out.println(enu.nextElement());
	}
