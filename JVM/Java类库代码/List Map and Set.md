# 结构的抽象接口们 --- List Map and Set

简单的看一下各个数据结构的抽象接口，一般说来就这么三个吧，比如之前看到的ArrayList和LinkedList是实现了List接口并且继承了AbstractList类，HashMap实现了Map接口并且继承了AbstractMap类，而HashSet实现了Set接口并且继承了AbstractSet类。

首先先从接口看起：

## List

先从List的性质看起，看看怎么区别其他两个接口的。这个接口的目的是可以使用**系数**来访问元素，并且可以对元素进行搜索。它和set最大的不同就是，它可以使得两个元素重复（a.equals(b)）。在注释中提示了一下，list中的equals和hashCode方法并不一定实现地很好。

所以对于一个List，它应该有如下函数声明：

* size() :表长
* isEmpty()：判空
* contains(Object o)：判是否存在
* Iterator<E> iterator()：定义一个迭代器
* [] toArray() : 化为数组
* add(E a)：添加元素,默认放在数组的最后
* remove(Object o) ：删除元素，默认删除最先碰到的相同元素
* containsAll
* addAll
* removeAll
* retainAll
* replaceAll
* clear()：清除所有元素
* equals：判断是否相等，判断的是这两个列表对象
* hashCode：计算hash值，在列表中计算hash都是使用如下方法：

```
int hashCode = 1;
for (E e : list)
    hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
```

* E set: 设置某个位置的值
* E get: 获取某个位置的值
* add(index，element)：在某个位置上添加一个元素
* remove(index)：删除某个位置上的元素
* indexOf(Object o)：返回的是第一次碰到这个o元素的下标。

那么抽象类AbstractList实现了这个接口，并对其中的函数做了一些实现，因此在ArrayList中就有很多方法直接使用了AbstractList中的实现方法。

那么就简单看几个方法来look一下：

```java
public boolean equals(Object o) { // 从这个方法中可以看到，最终比较两个对象是否相同就是比较它们的数组是否完全一样
    if (o == this)
        return true;
    if (!(o instanceof List))
        return false;

    ListIterator<E> e1 = listIterator();
    ListIterator<?> e2 = ((List<?>) o).listIterator();
    while (e1.hasNext() && e2.hasNext()) {
        E o1 = e1.next();
        Object o2 = e2.next();
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    return !(e1.hasNext() || e2.hasNext());
}
```

其他的和ArrayList中很相似吧，但是起始这个abstractList相当于是实现了一个List的骨架，所以后续在这个上面具体实现的一些列表结构根据这个骨架微调一下就可以了。

## Map

很显然这么一个结构是实现一个键值对的结构，并且一个map结构可以允许重复的键。但是每一个键只能映射一个值。map中很忌讳将一个键设置为一个可变的类型，所以它不允许使用map类型作为它的键，但是map类型本身可以用作value。最后，如果一些操作是会循环访问自身的，那么Map必须要处理这样的函数，比如抛出异常。

其中一些接口函数包括了：

* size() ：元素个数
* isEmpty()：是否为空
* containsKey(Object key)：是否存在某个键
* containsValue(Object value)：是否存在某个值
* get(Object key)：根据键获取某个值
* put(K key, V value)：根据某个键，添加或者修改某个键值对
* remove(Object key)：根据某个键删除某个键值对
* Set<K> keySet()：键的集合
* Collection<V> values()：值的集合
* Set<Map.Entry<K, V>> entrySet()：键值对的集合

同理，也能知道AbstractMap抽象类实现了Map接口的一个骨架，那么基本上其他的Map的基本方法也在这个基础上实现。

```java
public int hashCode() {
    int h = 0;
    Iterator<Entry<K,V>> i = entrySet().iterator();
    while (i.hasNext())
        h += i.next().hashCode();
    return h;
}

public int hashCode() {
    return (key   == null ? 0 :   key.hashCode()) ^
           (value == null ? 0 : value.hashCode());
}
```

## Set

那么也说过了，集合的实现是建立在没有两个重复元素的基础上的，也就是说在set中就不会出现有两个元素使得a.equals(b)，所以这里面也最多只有一个空元素。

同样对于集合而言，它很忌讳将一个可变的元素插入到集合中，尤其是禁止插入Set。

Set中的方法和List中的方法几乎是一模一样的，这里只着重讲几个：

* add：如果集合中已经有了要加的元素，那么就使得这个集合不变并且返回false

AbstractSet作为实现Set的一个抽象类，提供了实现Set的一个原型。

```java
public int hashCode() {
    int h = 0;
    Iterator<E> i = iterator();
    while (i.hasNext()) {
        E obj = i.next();
        if (obj != null)
            h += obj.hashCode();
    }
    return h;
}
```

