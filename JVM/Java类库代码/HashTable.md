# HashTable

在HashMap中也提到了这个hashtable哈希表的结构也是表示一对键值对，但是有一个前提就是这个键值对的键或者值不能为空。而对于它而言同样有两个参数制约着它：capacity和load factor。当插入元素的过程中，哈希表会查看是否有必要对当前的容量进行一个扩容。

最后提及一点是HashTable这个结构本身是一个线程安全的结构，如果你觉得线程安全并不一定需要保证，那就可以直接使用HashMap这个结构。

那么对于一个哈希表而言，并没有那么多花里胡哨的参数定义，其中包括的成员变量和HashMap类似：

* Entry<?, ?> table：存放Hash表元素的一个数组
* int count：现在哈希表中的元素个数
* int threshold：容量的阈值，如果超过这个大小就需要扩容。
* float loadFactor：哈希表的负载因子，阈值的计算仍然是capacity * loadfactor

## 初始化

初始化的操作对于threshold和loadfactor的初值有一个定义，同时对table进行一个初始化，这和HashMap的初始化就有一些不同了，我们回想一下在HashMap中对table成员变量进行初始化是出现在resize这个扩容函数中的，如果是初始化过程，则resize函数直接返回一个空的数组。这里直接就对数组进行了初始化。

```
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    table = new Entry<?,?>[initialCapacity];  // 数组变量的初始化，初始长度就是指定的容量
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}

public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

public Hashtable() {
    this(11, 0.75f);  // 从这里就可以看到了，如果调用默认的构造函数，那么默认的容量是11，而默认的负载因子是0.75
}  
```

## 搜索

这里的搜索把它看成两类，一类是判断当前的Hash表中存不存在某个键或者某个值

```
public boolean containsValue(Object value) {
    return contains(value);
}

public synchronized boolean contains(Object value) {
	// 如何查找里面是否有一个value，方法是从table中遍历每一个hash表入口，然后遍历入口中的每一根链表，查看是否有相同的value值
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    for (int i = tab.length ; i-- > 0 ;) {
        for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
            if (e.value.equals(value)) {
                return true;
            }
        }
    }
    return false;
}

public synchronized boolean containsKey(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length; // 这里是在处理入口的hash值
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) { 
    	// 直接遍历这个hash值上的链表，看是否有相同的键值，键值的比较就用key的哈希值
        if ((e.hash == hash) && e.key.equals(key)) {
            return true;
        }
    }
    return false;
}
```

那么其实获取结点的函数也是这么个做法：

```
public synchronized V get(Object key) { // 和containsKey的操作一摸一样，就只是返回的不一样而已
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

## 插入新结点

```
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {  // 在hashtable中value的值一定是非空的
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
    	// 下面是这样，现在找到了这个key值对应的表入口，那么此时就查看在这个入口中是否已经出现过这个键了，如果出现就直接替换该键值对的值。
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index); // 否则就需要将新的结点添加进去，但是此时已经知道是在哪个链表上了
    return null;
}

private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    if (count >= threshold) { // 如果此时添加的过程中已经让表格的大小超过了阈值，需要进行扩容
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table; // 扩容后我们重新获取一次表格和哈希值。
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];  // 此时再找到tab的入口处新建一个结点
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

在这里牵涉到了一个扩容操作rehash：

```
protected void rehash() {
    int oldCapacity = table.length; // 表的原来的长度
    Entry<?,?>[] oldMap = table; // 原来的表

    // overflow-conscious code
    int newCapacity = (oldCapacity << 1) + 1; // 新的容量设置为了老的容量的两倍呢
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];  // newMap就是指新的数组

    modCount++;
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;
	// 下面开始对老的数组中的元素进行一次重构， 过程就是遍历整个hash表，然后把每条链表上的元素进行一次重新hash的过程
    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index]; // 头插法
            newMap[index] = e;
        }
    }
}
```

## 删除一个结点

```
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {  // 此处相当于找到了该结点了
            modCount++;
            if (prev != null) {
                prev.next = e.next; // 修改这个结点的前向结点指针指向
            } else {
                tab[index] = e.next; // 否则该结点的下一个结点就应该称为链表头
            }
            count--;
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}
```

---

在这个HashTable中也实现了keySet，entrySet和values这三个集合，在这里就不赘述了。

---

讲到这里其实也可以结束了，因为主要的方法也已经讲完了，本质看了看HashTable就是一个HashMap的弱化版本，那么还有什么好说的呢？再说一个里面有的用到的结构吧。

在这个文件中藏着这样一个结构：**keys**，它的类型是enumeration类型，这是一个枚举接口，它能够枚举集合中的元素，但是如果看里面的功能的话其实和所谓的迭代器的作用几乎是一模一样的。枚举器是老式的接口慢慢被迭代器接口所取代。HashTable是相对较为老的一些接口，因此含延续使用着这个结构。

这个结构中应当要实现的方法只有两个：

* hasMoreElement：查看该集合中是否又更多的元素了。
* nextElement：获取集合中的下一个元素。

而keys同时实现了iterator这个接口是基于了这两个方法实现的。