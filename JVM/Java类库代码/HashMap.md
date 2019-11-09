# HashMap

hashmap本身是一个散列表的结构，它其实和早期的hashtable有一样的功能，只不过不同的在于它是线程不安全的并且允许键值为空。在hashmap的设计中有两个比较重要的参数：

* 容量：代表着在hash表中的入口数
* 负载因子：是衡量一个hash表有多满的一个重要指标，如果一个当前入口数已经超过了hash表的当前容量乘上负载因子，那么整个表就要重新hash。

所以我们可以看到在HashMap类中存在静态变量：

```C
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16  hash表初始容量
static final int MAXIMUM_CAPACITY = 1 << 30; // hash表最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f; // hash表初始负载因子
```

这里要提前说一下，在HashMap实现中有一点。Hashmap一般处于一个“桶状”的hash表状态，如果这个结构太大，即结点数过多时，会转换成树状结构，来提高性能。为什么不一开始就使用树状的HashMap呢？那是因为一个树结点比普通结点要大出一倍的空间，如果当前的hash表够用就没必要使用树状结点。当然如果入口数缩减到一定数量，HashMap也能保证可以从树状态变回到桶装结构。这需要两个阈值来控制：

```C
static final int TREEIFY_THRESHOLD = 8; // 从桶状态变为树状结构
static final int UNTREEIFY_THRESHOLD = 6; // 从树状结构转换为桶装结构
```

## 数据结构

在这个Map里面一共可能出现两种结点，要么是普通的桶结点，要么是树结点。

```C
static class Node<K,V> implements Map.Entry<K,V> {  // 这是普通的结点
        final int hash;  // 结点hash值
        final K key;  // 键
        V value;  // 值
        Node<K,V> next; // 指向下一个结点的引用

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() { // 有自己的一套hashcode的计算方法，key的hashcode异或上value的hashcode
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&  // 两个结点相同代表着key和value都相同
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

```C
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  // 树结点
        TreeNode<K,V> parent;  // 父节点指针
        TreeNode<K,V> left;    // 左孩子指针
        TreeNode<K,V> right;   // 右孩子指针
        TreeNode<K,V> prev;    // needed to unlink next upon deletion 在删除中要使用的前向指针
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        
    	// 暂时省略有关树的操作函数
        ...
}        
```

所以它有以下成员变量

* Node<K, V>[] table：也就是桶状态下的存放结点的一个hash表，它的形态整体是一个hash表，每个表项的结点通过内部的next指针连接。

* Set<Map.Entry<K, V>> entrySet：作为各个入口的一个缓存
* size：Map的大小
* threshold：意味着下一次重构时的大小，初始化时默认为DEFAULT_INITIAL_CAPACITY
* loadFactor：负载因子

## 初始化

这里面一共四个初始化的选项：

```C
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
    // 这一步得到的初始大小必然是2的n次方倍，因为该操作中是在通过位操作来保证末尾处为连续的1，然后加1

}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);  // 通过已有的Map对象来初始化当前的Map
}
```

``` java
final Node<K,V>[] resize() {  // 为什么把这个函数放在初始化中呢？因为在最初使用过程中table是空的，所以需要使用这个函数来初始化整个table，而后续使用中这个函数的作用是让table扩容
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {  // 当表非空
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  // 容量也变为原来的两倍
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold  // 简单的说表的扩容就是让整个表的阈值变为两倍
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;  // 初始化阶段表的初始容量应该就是老的阈值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {  
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;  // 修改当前的阈值
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  // 根据新的容量新开辟一个table
    table = newTab;// 老表格指向行表格
    if (oldTab != null) {  // 这里是只有在扩容的时候会做的，初始化到这里就结束了
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;  // 如果这个hash表的这个入口只有一个普通结点，那么只需要将这个结点重新放入新的hash表入口就可以了
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // 当前处于树状态就由树结点的处理函数处理
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {  // 这里开始搬移当前找到的这个结点后面的所有结点
                        next = e.next;
                        if ((e.hash & oldCap) == 0) { // 这里应该是在判断当前各个结点对于表格大小的位置限制
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {  // loTail就是原地操作
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) { // highTail是将原链表移动到加上了原来表长的位置处
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



## 插入一个结点

使用的是put方法，给出你要插入的键值对：

```C
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true); 
    // 这里小插一句，这个hash()方法是它的一个内部计算key值hash值得方法，其方法是让key求取hashcode然后和自身右移16位后异或
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;   // 如果tab长度为0，要进行初始化
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);  // 这是插入第一个结点，直接通过hash值计算得到下标，然后插入这个结点，newNode就是调用了构造方法
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;  // 如果发现这个新加入的结点的键已经存在于这个hash表中了，那么就获得这个结点
        else if (p instanceof TreeNode)  // 如果找到的这个结点是一个树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {// 走到这里并不一定是一个新节点呢
            for (int binCount = 0; ; ++binCount) { // 这个计数是在查看这个链是否已经过长
                if ((e = p.next) == null) {  // 这里就保证了这个结点一定是一个新节点了
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 如果长度过长，这个链表需要转换成树结构
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                 // 如果你从这个链表中找到了一个键相同的结点，那么直接拿得这个结点
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value; // 替换原来的值
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)  // 如果整个大小已经超过了阈值，需要对这个表格扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## 从Map中删除一个结点

再来看一下删除一个其中一个结点的操作：

```C
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;  // 很类似，如果直接找到了结点，那就获得该结点
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)  // 如果是树形态，则调用树结点的获取结点的方法
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {  // 从一根链表上查找对应的结点
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
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) // 如果这是一个树结点，那需要调用树中的删除函数
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)  // 如果删除的是链表头
                tab[index] = node.next;
            else  // 删除的是链表中的某个结点
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



## 从Map中获取一个结点

在使用Map时经常使用，由键值队的键去查询键值对的值，所以就是调用其中的get方法：

```
	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

其中最核心的就是getNode这个方法了，这个方法也用在了contains函数中（判断是否包含了该结点）

```java
	final Node<K,V> getNode(int hash, Object key) {  // 我感觉经过了添加和删除的函数的分析，这个函数的内容也没什么问题了
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

讲到contains操作，在Map中还加入了containsValue，你可以认为这两个操作一个是判断key值是否存在，一个是判断value值是否存在，在实现上也是很暴力的。

```java
public boolean containsValue(Object value) { // 但是这个函数看来非常奇怪，似乎忽略了树状态下的Map
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

## 其他

这里其实还有最后的几个操作，这些操作也是比较重要，渗透在不同地方吧。

我们看到在上面的加节点操作中，当单条链表的容量超过了阈值，那么此时就需要将链表改为一颗红黑树的结构了。

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {  // 找到该入口的结点
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);  // 这个函数的方法应该是将当前普通结点修改为一个树结点
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);  // 真正调用了treeify函数来对该链表进行重构
    }
}
```

---

## 三个常用的集合

下面来看三个常用的集合，当我们在编写程序的时候有很多时候想要获得这个Map中所有出现过的key值，所有出现过的value，或者获得这个Map中的所有元素。那么在本文件中，当然实现了这个三个集合。下面一个个来看：

### KeySet

```java
final class KeySet extends AbstractSet<K> {  
// 继承了Set的类，所以实现的是一个集合，具体集合是怎么实现的呢，这就要看具体的Set代码拉
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

### Value

```java
final class Values extends AbstractCollection<V> {  
// 应该比Set还要底层一点
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

### EntrySet

```java
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Map.Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

---

## TreeNode的各种操作