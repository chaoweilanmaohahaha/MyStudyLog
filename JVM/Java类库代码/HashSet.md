# HashSet

这个类是实现了Set这个接口，实际使用了哈希表来实现了集合，允许空元素进入集合。如果仔细地看里面的实现，你会发现属实就是一个HashMap的结构，但是这个类不仅完成了键值对映射，这个主要的目的还是实现集合。还是先看一下成员变量：

* HashMap<E, Object> map：使用一个map来存放集合中的元素
* Object PRESENT：这是一个后备成员，为什么要用这个是因为它只想使用Map中的键来存用户想要操作的数据，而并不用到map中的value，所以使用一个假的value值就行了。

## 初始化

我们来看一下HashSet在初始化的时候的操作，你就知道HashSet和HashMap有什么联系了。

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
} 

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
} // 几乎所有的初始化的操作都是借助了hashmap的操作
```

## 具体实现

### 增添结点

```
public boolean add(E e) {
    return map.put(e, PRESENT)==null; 
    // 添加结点其实只考虑Map中的key，你这样发现在HashMap中的特性保证了，添加的这个元素不会重复。
}
```

### 删除结点

```
public boolean remove(Object o) {
    return map.remove(o)==PRESENT; 
    // 这里体现一个问题，为什么不直接插入一个null，而要实例化一个object出来，原因就是在这里
}
```

其他简单的操作包括contains，size等都借助了map中的方法实现了，所以说HashSet是在HashMap的基础上实现的。

