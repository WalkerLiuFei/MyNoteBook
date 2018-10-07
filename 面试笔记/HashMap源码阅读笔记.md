---
title: HashMap源码阅读笔记
---

HashMap 是最常用的集合类之一。
 
# HashMap概况 

## HashMap的存储结构 
+ 整体是以单链表的形式链接起来的
+ Key是以数组的形式存储。
+ Value 是以单链表的形式存储
 
这样的存储结构结构保证了增删改查时间复杂度的都可以是O（1）常数级别。

## 关于HashMap的一些小细节
 
+  **你知道get和put的原理吗？equals()和hashCode()的都有什么作用？**
 + 通过对key的hashCode()进行hashing，并计算下标( n-1 & hash)，从而获得buckets的位置。如果产生碰撞，则利用key.equals()方法去链表或树中去查找对应的节点
+ **如果两个Key的hash发生碰撞，但是equals方法没有相等，会发生什么？**

```Java
	
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//通过hash计算出来的下标没有node的话，那么就把这个node填充
        if ((p = tab[i = (n - 1) & hash]) == null) 
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
			//如果hash和equals计算均相等。会提换原来的元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
				
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
					 	//如果hash值计算的相同，但是equals计算的不同，则会放到链表的最后
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
			
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
``` 
 通过上面的put方法的具体实现，也可以看出，为什么HashMap的Key为什么要同时重写hashCode和equlas方法。同样的，你可以查看get方法的具体实现。其实这样是一个对Hash冲突的解决办法。

+ **HashMap中对Key的 HashCode的处理？**
 
 + 首先看一段代码

	```Java
		
		static final int hash(Object key) {
	        int h;
	        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	    }
	
	```
 这段代码的意思是，高位和低位进行扰动计算。

 + **为什么进行扰动计算？** 首先，扰动计算后的代码HashMap是一个Hash桶加上链表。在用过key计算出的Hash值，我们知道是一个 32位的值。但是我们需要的hash值有效长度没有那么多。可能只有hashMap。key数组那么长的长度即可。但是带来的问题是，我们key的hash值在length的范围内很有可能不是 均匀分布的。很有可能带来hash冲突。
 [查看知乎上的回答](https://www.zhihu.com/question/51784530)
 + **HashTable 的Key为什么不能为null**
	 + 看HashTable源代码的put方法实现。可以看key的hash计算方法。当然会出现空指针错误了

```Java
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
```

----未完待续----