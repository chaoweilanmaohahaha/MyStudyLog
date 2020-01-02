# PriorityQueue

优先队列是基于堆结构来实现的，它们在堆上的排列顺序依靠着对象的比较器也就是Comparable。队列上不可以存在空。在jdk的实现中，该优先队列的实现是一个小顶堆，如果多于一个最小值就只取其中的一个就行了。最后提一手优先队列是线程不安全。看一手其中的成员变量：

* final int DEFAULT_INITIAL_CAPACITY = 11：默认初始化时数组的容量；

* Object[] queue：用来存放堆内对象的数组；
* int size：指队列中包含的对象数
* final Comparator<? super E> comparator：优先队列中的比较器，如果这个比较器为空，则就按照元素默认的顺序来比较，如果初始化时要求了比较器，则该队列按照这个比较器来比较顺序。

## 初始化

讲道理优先队列的初始化的方法有点多，因为按照不同的要求初始化可能性比较多：

```java
public PriorityQueue() {  // 默认方法
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityQueue(int initialCapacity) { // 指定默认容量
    this(initialCapacity, null);
}

public PriorityQueue(Comparator<? super E> comparator) { // 指定比较器
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}

public PriorityQueue(int initialCapacity, Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity]; // 这个初始化就初始化了数组
    this.comparator = comparator;
}

public PriorityQueue(Collection<? extends E> c) { // 使用一个集合初始化这个队列
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        initElementsFromCollection(ss);
    }
    else if (c instanceof PriorityQueue<?>) { 
        PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        initFromPriorityQueue(pq);
    }
    else {
        this.comparator = null;
        initFromCollection(c);
    }
} // 当然在这个文件中实现了从不同的集合中把数据读到这个队列中的方法，那么如果原本的数据并没有按照堆的排列方法，则需要将这个结构化为堆
```

## 堆操作

我们先从这个基本的操作开始，来一步步看这个优先队列的其他基本操作，在初始化中如果从一个普通的集合中获取数据，那么就需要将数据堆化，这也是初始化的一个部分：

```java
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--) // 找最后一个父节点
        siftDown(i, (E) queue[i]); // 从这个结点开始进行下筛
}

private void siftDown(int k, E x) {
    if (comparator != null) // 如果指定了队列的比较器，那么就按照该比较器来进行比较，从而进行下筛
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x); // 没有指定的话就按照默认顺序比较
}

private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1; // 最后一个父节点
    while (k < half) {
        int child = (k << 1) + 1; // 左孩子的位置
        Object c = queue[child];
        int right = child + 1; // 右孩子的位置
        if (right < size && 
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right]; // 这一步是在比较两个孩子之中那个比较小的
        if (comparator.compare(x, (E) c) <= 0) // 如果父节点比孩子节点都小就不动，并且堆已经完毕
            break;
        queue[k] = c;  //否则更新父节点，为什么这边不立刻做交换呢，你可以试一下发现不立刻做交换也可以保证大小
        k = child;
    }
    queue[k] = x;
}
```

那么有了下筛的操作就有上筛：

```java
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;  // 父节点的下标
        Object e = queue[parent]; 
        if (comparator.compare(x, (E) e) >= 0) // 当前的结点如果比父节点大则不用再上筛了
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
```

## 增加元素

```java
public boolean add(E e) {
    return offer(e);
}

public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1); 
        // 如果超长了就需要进行一次扩容，扩容的操作其实和arraylist非常相似，只不过给出下一个长度不一样，在这里，如果比长度比64小则每次扩大两倍，否则扩大3/2
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e); // 插入该元素并进行上筛
    return true;
}
```

## 删除一个元素

```java
public boolean remove(Object o) { 
// 还有另一个版本叫removeEq，不过因为最终调用的函数是相同的，所以只要看一下这个方法就行了
    int i = indexOf(o);
    if (i == -1)
        return false;
    else {
        removeAt(i);
        return true;
    }
}

private E removeAt(int i) {
    // assert i >= 0 && i < size;
    modCount++;
    int s = --size;
    if (s == i) // removed last element  // 删除最后一个元素一定对堆的形状没有影响
        queue[i] = null;
    else {
        E moved = (E) queue[s];  // 拿到最后一个元素
        queue[s] = null; // 清除最后一个元素
        // 下面的操作是整个堆中最为核心的方法，一共包含了三种情况
        siftDown(i, moved);  // 最基本的，我把一个元素从叶子结点移动上来之后，最可能需要下调这个结点值
        if (queue[i] == moved) {
            siftUp(i, moved);  // 但是可能这个叶子值并不需要下调，则考虑，因为这类似往第i个位置插入了一个元素，因此上调。
            if (queue[i] != moved) // 但是如果上调成功了，就返回当前上调了的元素
                return moved;
        }
    }
    return null; // 返回空只有可能是下调成功了或者调动元素依旧在原地没有移动
} // 为什么这么设计，我们在下面看迭代器时详细看一下
```

```java
public E poll() { // 弹出了第一个元素，然后将最后一个元素替换上来之后只要进行下筛的操作就行了
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    return result;
}
```

---

从这里简单看一下iterator，为什么需要看iterator的实现，是因为其中的实现不是简单的调用优先队列中的函数，而是有它自己的设计

```java
private final class Itr implements Iterator<E> {

    private int cursor = 0; // 迭代器指针

    private int lastRet = -1; // 记录指针的上一个位置

    private ArrayDeque<E> forgetMeNot = null; // 后备队列

    private E lastRetElt = null;

    private int expectedModCount = modCount;

    public boolean hasNext() {
    	// 判断是否还有元素的方法是看指针是否指到了底或者后备队列中有没有元素了
        return cursor < size ||
            (forgetMeNot != null && !forgetMeNot.isEmpty());
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (expectedModCount != modCount)
            throw new ConcurrentModificationException();
        if (cursor < size)
            return (E) queue[lastRet = cursor++]; // 就算是在访问上调了的元素，cursor也依旧加1了
        if (forgetMeNot != null) { // 注意此时后备队列有值，说明真正下一个位置需要访问的应该是后备队列中的元素
            lastRet = -1;  // 保证了我如果下一次删除删除的是这个上调元素，那么就直接去查找这个元素就行了
            lastRetElt = forgetMeNot.poll();
            if (lastRetElt != null)
                return lastRetElt;
        }
        throw new NoSuchElementException();
    }

    public void remove() {
        if (expectedModCount != modCount)
            throw new ConcurrentModificationException();
        if (lastRet != -1) {
            E moved = PriorityQueue.this.removeAt(lastRet);
            lastRet = -1; // 当我们完成一次删除后，就不允许第二次删除了
            if (moved == null)  // 从这里对应上面的removeAt函数来看，如果最终下调或者不动，那么指针向后走一格，正好下一个访问的就是被调动的元素
                cursor--;
            else { // 那么如果被上调了呢，这样会导致调动元素无法被获取具体的位置，指针也不能向前移动，那干脆把调动的元素记录下来
                if (forgetMeNot == null)
                    forgetMeNot = new ArrayDeque<>();
                forgetMeNot.add(moved);
            }
        } else if (lastRetElt != null) { // 这里的情况是，我现在遍历到已经被上调的元素，那么此时并不需要移动指针，要做的是从堆中删除这个上调元素
            PriorityQueue.this.removeEq(lastRetElt);
            lastRetElt = null;
        } else {
            throw new IllegalStateException();
        }
        expectedModCount = modCount;
    }
}
```

