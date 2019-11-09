# Stack and Queue

## Queue

在java中队列是一个接口，**接口本身是不能被实例化的**，所以你在使用的过程中没有出现过如下的代码：

```
Queue<Integer> q = new Queue<>();
```

但是接口接口可以指向实例的类对象，比如我们真正在使用的时候一般使用linkedList

```
Queue<Integer> q = new LinkedList<>();
```

而确实在LinkedList中实现了队列的所有方法（其实是实现了Deque，而Deque继承了Queue），所以在这里以LinkedList中的实现来看一下队列的实现。在Queue接口中一共要实现如下一些方法：

* add: 往队列中加入一个元素
* offer：同样是往队列中加入元素，不过如果当队列的容量是有限的时候，那么这个方法要优于add
* remove：移除头部一个元素，如果队列空报异常
* poll：移除头部的元素， 如果队列空返回NULL
* element：获取队列头部的元素，如果队列空报异常
* peek：获取队列头部的元素，如果队列空返回NULL

提一嘴，Queue是借助linkedlist实现的话，它是线程不安全的

```
// Queue operations.

    /**
     * Retrieves, but does not remove, the head (first element) of this list.
     *
     * @return the head of this list, or {@code null} if this list is empty
     * @since 1.5
     */
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

    /**
     * Retrieves, but does not remove, the head (first element) of this list.
     *
     * @return the head of this list
     * @throws NoSuchElementException if this list is empty
     * @since 1.5
     */
    public E element() {
        return getFirst(); // 如果对象为空会报NoSuchElementException
    }

    /**
     * Retrieves and removes the head (first element) of this list.
     *
     * @return the head of this list, or {@code null} if this list is empty
     * @since 1.5
     */
    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    /**
     * Retrieves and removes the head (first element) of this list.
     *
     * @return the head of this list
     * @throws NoSuchElementException if this list is empty
     * @since 1.5
     */
    public E remove() {
        return removeFirst();  // 如果对象为空会报NoSuchElementException
    }

    /**
     * Adds the specified element as the tail (last element) of this list.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Queue#offer})
     * @since 1.5
     */
    public boolean offer(E e) {
        return add(e);
    }
```

---

## Stack

相比于队列，栈并不是一个接口，而是一个类。但是这个类继承了在类库中另一个比较庞大的常用的类vector，所以它的实现在很大程度上接住了vector这个父类，包括其中使用的成员以及调用的方法，如果想要完全看懂Stack的实现，还是要看一下vector这个类，注意，Stack是线程安全的，因为vector本身也是线程安全的。

不过我们还是稍微看一下具体这个类里做了什么，首先先要说明这个类中没有任何成员。

```
	public E push(E item) {
        addElement(item); // 调用了vector中的函数，将对应的对象加到了elementData尾部

        return item;
    }
    
    public synchronized E pop() {
        E       obj;
        int     len = size(); // 获取elementData的总长度
 
        obj = peek();  // 获得elementData的最后一个元素，数组如果是空的那就会报出异常
        removeElementAt(len - 1); // 删除最后一个元素

        return obj;
    }
    
    public synchronized E peek() {
        int     len = size();

        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);  // 返回最后一个元素
    }
    
    public synchronized int search(Object o) {  // 查找栈中的系数
        int i = lastIndexOf(o);

        if (i >= 0) {
            return size() - i;  // 返回该元素到栈顶的距离
        }
        return -1;
    }
```

---

# Deque

双端队列，这个接口继承了queue接口，是一种特殊的队列结构。顾名思义，双端队列就是从两端都能加入元素，并且在两端都能删除元素。所以这个结构非常灵活，可以用它实现一个队列，也可以用它来实现一个栈。在linkedList中有实现了Deque的所有接口函数，其中包括了常用的：

* addFirst
* offerFirst
* addLast
* offerLast
* removeFirst
* pollFirst
* removeLast
* pollLast
* getFirst
* peekFirst
* getLast
* peekLast

这些方法都可以参照Queue中方法的介绍想出具体的使用。