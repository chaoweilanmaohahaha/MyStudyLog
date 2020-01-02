# LinkedList

这个就是我们常说的链表结构，这个类的实现继承了AbstractSequentialList，并且实现了List和Deque等接口，它本身是线程不安全的。在这个类中有三个成员：

* size: 记录了这个链表中元素的个数
* first节点： 指向第一个元素的结点
* last节点： 指向最后一个元素的结点

我们先来看一下代表这个节点的结构，在这个文件中单独定义了私有类Node，而first和last就是该类的实例：

```
	private static class Node<E> {
        E item;  // 节点中存放的元素
        Node<E> next;  // 后向指针
        Node<E> prev;  // 前向指针
 	// 所以这个结构就是一个双向链表
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

LinkedList的初始化并没有什么可以讲的，着重看一下里面所包含的方法以及实现。

### 添加元素

首先看一下往这个链表里面加元素是怎么添加的：

```
	public boolean add(E e) {
        linkLast(e);   // 默认添加到链表尾部
        return true;
    }
    
    void linkLast(E e) {
        final Node<E> l = last; // 获得当前的链表尾节点
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode; // 移动尾节点
        if (l == null)
            first = newNode;  // 如果这是加进去的第一个节点，那么头节点和尾节点都指向这个节点
        else
            l.next = newNode; // 否则要让前一个结点的next指针指向该节点
        size++;
        modCount++;
    }
```

那么如果就是想让该元素加入到链表头部呢？

```
	public void addFirst(E e) {
        linkFirst(e);
    }
    
    private void linkFirst(E e) { // 对比linkLast函数，实现是一样的
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;  // 移动first结点指针
        if (f == null)
            last = newNode; // 如果这是第一个元素那么就让尾指针和头指针都指向这个结点
        else
            f.prev = newNode; // 否则让后一结点的前向指针指向该结点
        size++;
        modCount++;
    }
```

那如果要在链表中的某一个特定的位置加入某个元素：

```
	public void add(int index, E element) {
        checkPositionIndex(index);  // 这里确认过了index的位置是否合法

        if (index == size)  // 如果此时给出的系数和当前元素个数相同，那就说明就是插入到尾部
            linkLast(element);
        else
            linkBefore(element, node(index));  // 否则使用linkBefore这个函数，node(index)先理解为找到了特定系数的结点，因为确认过了index的合法性，所以保证node取出的结点非空
    }
    
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev; // 获取succ结点的前一节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        // 将该节点的前向指针指向succ的前一节点，而后向指针指向succ
        succ.prev = newNode; // 修改succ的前向指针，指向新节点
        if (pred == null)
            first = newNode;  // 如果当前的这个结点就是头节点了，那么就要修改first指针
        else
            pred.next = newNode;  // 否则需要让原来succ的前向结点指向该结点
        size++;
        modCount++;
    }
```

### 查找元素

可以看到在add群函数的最后用到了node函数去查询某个index系数对应的结点，那么这里就有必要看一下具体在链表中是怎么样查找指定结点的了。

```
	public E get(int index) {
        checkElementIndex(index);
        return node(index).item;  // 也是使用了node函数去获取了指定的结点
    }
    
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);  // 使用了node函数获取指定的结点
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
    
    Node<E> node(int index) {  // 每一次调用这个函数的时候都需要check一次index是否在查找范围内，这样保证必然能够查找到结点
        // assert isElementIndex(index);
		// 一下查找的方法根据链表长度作了优化，size >> 1指的是链表长度的一半
        if (index < (size >> 1)) {   // 如果当前查找的在链表的前半段，那么就从头部查找
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {  // 如果在链表的后半段，就从尾部向前查找
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

但是针对indexOf来说，它是根据对象来查找这个对象在这个链表的具体系数，那么此时必须要遍历整个链表，包括lastindexOf。

### 删除元素

最后讲讲如何从这个链表中删除元素吧，首先还是要实现接口指定的remove函数：

```
	public E remove(int index) {
        checkElementIndex(index);  // 同样保证了系数的合法性
        return unlink(node(index));
    }
    
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;  // 在删除之前获取删除结点的前向结点和后向结点
        final Node<E> prev = x.prev;

        // 修改前向结点的指针，注意如果删除的这个结点已经是第一个了，那么需要修改首指针，否则就要让前向指针的下一项指向后向结点
        if (prev == null) {  
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

		// 同理处理后向结点
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

其实上面的情况已经包括了删除结点的所有细节了，那么像removeFirst和removeLast就是一个道理了，最多看一下clear这个清空操作：

```
	public void clear() {
        // Clearing all of the links between nodes is "unnecessary", but:
        // - helps a generational GC if the discarded nodes inhabit
        //   more than one generation
        // - is sure to free memory even if there is a reachable Iterator
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
```

其实下面有关iterator和listIterator的问题可以参照arraylist中的理解，迭代器中调用了linkedlist中的方法，来实现了链表的迭代器。

---

这里补充说一下，这个链表的作用不只是可以实例化一个LinkedList类，在之后使用学习Queue(队列)， Deque(双端队列)时，你会发现它们在使用的时候是实例化为Linkedlist，所以在这个文件中，也实现了queue和deque中需要使用的接口函数，在介绍队列和双端队列的时候再看吧！

---

## 总结：

1、LinkedList有哪些优势？

​	插入和删除效率高，只需要插入某个结点，不需要做搬移动作；并且作为链表来说，不存在扩容这个说法的。

2、LinkedList有哪些劣势？

​	搜索和修改某个特定位置数据效率底，每次都需要遍历一下链表。并且对于每个元素而言都需要分配一个Node结构，增加了额外的空间开销。

