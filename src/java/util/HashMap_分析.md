# HashMap

众所周知 HashMap 底层是基于 `数组 + 链表` 组成的，不过在 jdk1.7 和 1.8 中具体实现稍有不同。

## Base 1.7

### hashmap1.7 结构图

<img src="https://github.com/muyutingfeng/jdk1.8-source-analysis/raw/master/note/doc/java.util.HashMap_hashmap_internal_structure.png?raw=true" alt="java.util.HashMap_hashmap internal structure" style="zoom: 50%;" />









## Base 1.8

当 Hash 冲突严重时，在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为 `O(N)`。

因此 1.8 中重点优化了这个查询效率。

### hashmap1.8 结构图

<img src="https://github.com/muyutingfeng/jdk1.8-source-analysis/raw/master/note/doc/java.util.HashMap_hashmap_base_on_jdk8.png?raw=true" alt="java.util.HashMap_hashmap base on jdk1.8.png" style="zoom:50%;" />

### 核心成员变量

```java
    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     *
     * 用于判断是否需要将链表转换为红黑树的阈值
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     *
     * HashEntry 修改为 Node。
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
```



### put方法

```java
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //1.判断当前桶是否为空，空的就需要初始化（resize 中会判断是否进行初始化）。
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //2.根据当前 key 的 hashcode 定位到具体的桶中并判断是否为空，为空表明没有 Hash 冲突就直接在当前位置创建一个新桶即可。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //3.如果当前桶有值（ Hash 冲突），那么就要比较当前桶中的 key、key 的 hashcode 与写入的 key 是否相等，相等就赋值给 e,在第 8 步的时候会统一进行赋值及返回。
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //4.如果当前桶为红黑树，那就要按照红黑树的方式写入数据。
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //5.如果是个链表，就需要将当前的 key、value 封装成一个新节点写入到当前桶的后面（形成链表）。
                        p.next = newNode(hash, key, value, null);
                        //6.接着判断当前链表的大小是否大于预设的阈值，大于时就要转换为红黑树。
                        // -1 for 1st
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    //7.如果在遍历过程中找到 key 相同时直接退出遍历。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //8.如果 e != null 就相当于存在相同的 key,那就需要将值覆盖。
            // existing mapping for key
            if (e != null) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //9.最后判断是否需要进行扩容。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### get方法

```java
		public V get(Object key) {
        Node<K,V> e;
        //1.将key hash之后取得所定位的桶。
        //2.如果桶为空则直接返回 null 。
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

```java
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
            // always check first node
            //3.否则判断桶的第一个位置(有可能是链表、红黑树)的 key 是否为查询的 key，是就直接返回 value。
            if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //如果第一个不匹配，则判断它的下一个是红黑树还是链表。
            if ((e = first.next) != null) {
                //红黑树就按照树的查找方式返回值。
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //不然就按照链表的方式遍历匹配返回值。
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

从这两个核心方法（get/put）可以看出 1.8 中对大链表做了优化，修改为红黑树之后查询效率直接提高到了 `O(logn)`。

但是 HashMap 原有的问题也都存在，比如在并发场景下使用时容易出现死循环。

```java
final HashMap<String, String> map = new HashMap<String, String>();
for (int i = 0; i < 1000; i++) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            map.put(UUID.randomUUID().toString(), "");
        }
    }).start();
}
```

看过上文的还记得在 HashMap 扩容的时候会调用 `resize()` 方法，就是这里的并发操作容易在一个桶上形成环形链表；这样当获取一个不存在的 key 时，计算出的 index 正好是环形链表的下标就会出现死循环。



<img src="https://github.com/muyutingfeng/jdk1.8-source-analysis/raw/master/note/doc/java.util.hashmap_%E7%8E%AF%E5%BD%A2%E9%93%BE%E8%A1%A81.jpg?raw=true" alt="环形链表" style="zoom:50%;" />



![环形链表2](https://github.com/muyutingfeng/jdk1.8-source-analysis/raw/master/note/doc/java.util.hashmap_%E7%8E%AF%E5%BD%A2%E9%93%BE%E8%A1%A82.jpg?raw=true)



### 遍历方式

还有一个值得注意的是 HashMap 的遍历方式，通常有以下几种：

```java
Iterator<Map.Entry<String, Integer>> entryIterator = map.entrySet().iterator();
        while (entryIterator.hasNext()) {
            Map.Entry<String, Integer> next = entryIterator.next();
            System.out.println("key=" + next.getKey() + " value=" + next.getValue());
        }
        
Iterator<String> iterator = map.keySet().iterator();
        while (iterator.hasNext()){
            String key = iterator.next();
            System.out.println("key=" + key + " value=" + map.get(key));
        }
```



`强烈建议`使用第一种 EntrySet 进行遍历。

第一种可以把 key value 同时取出，第二种还得需要通过 key 取一次 value，效率较低。

> 简单总结下 HashMap：无论是 1.7 还是 1.8 其实都能看出 JDK 没有对它做任何的同步操作，所以并发会出问题，甚至 1.7 中出现死循环导致系统不可用（1.8 已经修复死循环问题）。

因此 JDK 推出了专项专用的 ConcurrentHashMap ，该类位于 `java.util.concurrent` 包下，专门用于解决并发问题。





1. put(K key, V value)
2. putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)
3. resize
4. get
5. remove
6. replace



