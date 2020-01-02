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

这里要提前说一下，在HashMap实现中有一点。Hashmap一般处于一个“桶状”的hash表状态，如果这个结构太大，即结点数过多时，会转换成树状结构，来提高性能（其实后面可以看到这个转换并没有删除原来的桶结构）。为什么不一开始就使用树状的HashMap呢？那是因为一个树结点比普通结点要大出一倍的空间，如果当前的hash表够用就没必要使用树状结点。当然如果入口数缩减到一定数量，HashMap也能保证可以从树状态变回到桶装结构。这需要两个阈值来控制：

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
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // 当前处于树状态就由树结点的处理函数处理，为什么下面的搬动过程中会出现高端链表和低端链表两条链表，那是因为在这个split的过程中会出现裁剪的行为，具体看一下split的操作
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
public boolean containsValue(Object value) {
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

我们仔细看一下TreeNode类可以看到，它是继承了LinkedTreeMap类的Entry类，但是如果再深入下去，我们又会发现这个Entry类同时又继承了HashMap的Node类。所以相当于这个TreeNode存在两套定义，一个可以作为树结点被红黑树处理，但是它又还是Node结点挂在了table表上，因此实际上TreeNode的存在只是为了提高查找效率。

所以我们发现在contains函数，包括树结构的遍历器中只提及了table表。

下面看一下具体的jdk中的红黑树写法，也算是学习一下：

### 找到一个结点

```java
final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
    // 这个函数很浓缩，判断是否是从中间节点还是从根结点
}

final TreeNode<K,V> find(int h, Object k, Class<?> kc) {  
// 已经保证了一定是从根结点出发的， h是hash值， k是键
// 这个的本质还是一个迭代，只不过分成按照hash值和按照类来查找
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))  // 找到了结点
            return p;
        else if (pl == null)  // 走到这里，说明hash值相等，但是并不是想要找的那个结点
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                  // 获取的是kc的类名已经接口名，例class M implement K
                 (dir = compareComparables(kc, k, pk)) != 0)
                 // 看k和pk进行比较大小的一个函数，这是用类来进行比较
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}
```

### 化为树形态和还原

```java
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    // 下面那个x默认是表头指针
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) { // 如果这是根节点
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) { // 下面开始查找插入位置
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk); 
                    // 走到这里说明hash值相等了，那就要使用这个函数来比较这两个结点的大小

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x); // 插入一个结点之后用这个函数来维持红黑树
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);  // 这样做了需要保证一点，链表头就要是树的根节点了
}

static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        if (root != first) {
            Node<K,V> rn;
            tab[index] = root;
            TreeNode<K,V> rp = root.prev; // 先获取这个结点的前向结点
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp; // 让root结点的下一个结点的前向指针指向rn
            if (rp != null)
                rp.next = rn; //  修改指针
            if (first != null)
                first.prev = root;  // 修改指针
            root.next = first;
            root.prev = null;
        } // 这一步究竟在干啥，其实就是将root的位置移动到了表头之后，需要修改root的一些指针指向，同时又要修改原表头结点的指针指向
        assert checkInvariants(root);
    }
```

那么解树结构的代码就相对简单了很多：

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);  // 还原成Node结点
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    } // 然后此时根据结点next顺序重新构建链表
    return hd;
}
```

### 插入与删除

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {  // 用在之前的那个putVal中
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;  // 从根结点处开始查找
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {  // 一样道理，到这边说明hash值已经是一样的了
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        } // 走到这里，说明一点，那就是如果这个key已经出现在这棵树当中了，那么我们直接把这个结点返回出去
		// 那从下面开始就是没有找到该结点位置的操作了
        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0) // 根据上面dir最后的结果确定新结点的插入点
                xp.left = x;
            else
                xp.right = x;
            xp.next = x; // 修改指针
            x.parent = x.prev = xp;  // 修改指针
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x)); // 插入这个新节点后要重新调整结构
            return null;
        }
    }
}

final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {  // 在上面的remove操作中使用
    //从这个参数列表中也可以看出，这个删除的操作需要从两个地方都删除掉
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    if (pred == null) // 当前结点的前向结点是空的
        tab[index] = first = succ; // 直接让表头指向它的next结点
    else
        pred.next = succ; // 它的前向结点的后继应该是自己的后继
    if (succ != null)
        succ.prev = pred; // 调整后继结点的前向指针
    if (first == null)
        return;
    if (root.parent != null)
        root = root.root();
    if (root == null || root.right == null ||
        (rl = root.left) == null || rl.left == null) {
        tab[index] = first.untreeify(map);  // too small
        return;
    } // 这里以上可以认为是在调整链表的指针情况
    // 从下面开始是调整红黑树上删除了该节点的操作，由于删除一个结点之后树的颜色和结构都要发生变化，这部分我们留到数据结构中去探讨，此处只需要知道的是，这里是在处理删除红黑树的结点所必须的一些操作
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        TreeNode<K,V> s = pr, sl;
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        TreeNode<K,V> sr = s.right;
        TreeNode<K,V> pp = p.parent;
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        }
        else {
            TreeNode<K,V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;
    if (replacement != p) {
        TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    if (replacement == p) {  // detach
        TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        moveRootToFront(tab, r);
}
```

### 剪枝

这是我们看到的有关树结点的最后一个操作，也就是树的裁剪，这一部分和上面的resize的一些操作密切相关，考虑到树的结构可能也会非常大，所以要适当对树结点进行裁剪以保证效率。这个函数在resize函数中调用了，这也解释了为什么在resize的扩容操作中，链表会分为低端链表和高端链表两条。

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
    // bit指老的table的长度
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        // 这里开始判断把这个树结点归为哪一个部分比较合适，分界线是老的表长度
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD) // 如果此时裁剪过后发现低端的链表长度比较短，则可以将树结构释放掉
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead; // 否则
            if (hiHead != null) // (else is a lready treeified)
                loHead.treeify(tab);  // 重新建树
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD) // 同样的操作
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

到这里，以上就是hashmap中常见的一些方法了，而文件到这里其实还并没有结束，因为使用了红黑树的结构，所以余下部分都是有关红黑树的操作了，这些操作的详细解释留到数据结构中再来详细叙述了。

---

## 总结

1、Hashmap扩容

​	简单的说就是hashmap的resize()方法具体到底怎么做的。首先我们先要知道扩容的大小以及什么时候扩容，根据tableforsize我们知道每次扩容的大小一定保证是2的n次幂，这个2的n次幂要保证正好比当前的长度要长，之所以这样安排还是和下面的hash有关。那么当整个表的容量已经大于threshold的时候，就需要进行扩容。

​	那么扩容的第一步就是将整个表长增加，那么第二步需要做的是重新散列表中元素。根据hash的法则：hash & (n-1)，再根据前面的大小是2的n次幂，我们会发现实际上这是屏蔽了n-1以上的位数，因为n-1的形式必然是全0然后全1。这样当我表长扩大了，那就相当于要少屏蔽一位，自然需要重新hash了。所以根据多出来的一位，链表就可能分成两条链。同样树结构因此也需要做一次剪枝的过程，并且剪枝的过程是重新hash后去构造新的链表。那么在构造完后依旧需要检查是否需要树化。

2、最小树化容量 ： 64？

3、hash冲突的解决
	采用的是链地址法，将有hash冲突的挂在了一根链表上

4、为什么树化大小阈值是8，不是4？

​	首先在源码的注释中讲到，根据泊松分布计算，在hash离散型很好的情况下，可以保证链表长度大于8的情况发生的概率是亿分之6，也就是基本不会发生。但是为了防止一些实现时不好的散列还是需要把这种情况包含进去。其次树化本身是一件开销很大的事，无论是对于存储空间而言还是执行效率而言。那么反过来分析你会发现，在长度为8的时候，树的查找效率和链表的查找平均效率也差不多，因此没必要在4的时候直接树化了。