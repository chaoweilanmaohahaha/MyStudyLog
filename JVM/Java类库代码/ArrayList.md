# ArrayList

ArrayList这个类继承了AbstractList的抽象类，实现了List接口。这个数据结构就是一个列表类型，实现不是很复杂，下面来看一下具体的实现细节，首先要知道一下类内成员：

* DEFAULT_CAPACITY :  list默认初始化时期的容量。
* EMPTY_ELEMENTDATA ： 空对象使用的实例。
* DEFAULTCAPACITY_EMPTY_ELEMENTDATA ： 初始化空对象使用的实例。
* elementData ： 类型是Object类型，存放指定的元素，修饰符是transient
* size ： 当前list元素的数目

ArrayList本身是线程不安全的。

---

### 初始化

ArrayList中一共有三种初始化的方法：对应的例子也放在旁边

```
	public ArrayList(int initialCapacity) {   // e.g. new ArrayList<>(5)
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
	}
	
	public ArrayList() { //  new ArrayList<>();
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    public ArrayList(Collection<? extends E> c) {  // e.g ArrayList(int a[])
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;  // 和直接初始化为空还不太一样
        }
    }
```

### IndexOf

首先对于一个列表来说，有一个写好的方法来判断是否存在某个对象contains:

```
	public boolean contains(Object o) {
        return indexOf(o) >= 0; 
    }
```

使用的就是indexOf，那么indexOf所作的就是从列表中查找某个对象，返回它的索引：

```
	public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

可以看到的是在arraylist中，所谓查找的方法，就是遍历这个数组，然后查找是否存在该元素，返回第一个出现该元素的位置，简单粗暴！和它一样还实现了一个lastindexof的方法，用来从末尾开始查找。

### add and remove

比较核心的方法，往一个表中加入一个元素或者删除一个元素，先来看一下具体的实现：

```
	public boolean add(E e) {  // 默认插入到列表尾部
		// 为了保证一定有空间装下新元素，要增大空间
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    public void add(int index, E element) {
        rangeCheckForAdd(index);  //检查插入的范围

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1, size - index);
        //增加的方法就是将右边的元素都向右移动一个单位

        elementData[index] = element;
        size++;
    }
```

```
	public E remove(int index) {  // 按下标删除
        rangeCheck(index);  // 这个需要进行区域合法查询

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        // 删除了中间某个元素后需要将后面的元素统一向前移动
                             
        elementData[--size] = null; // clear to let GC do its work  //细节

        return oldValue; //返回值是删除的元素
    }
    
    public boolean remove(Object o) { // 按对象删除
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);  
                    // 这个代码和remove方法是一样的，只是缺少了合法性验证，这个循环内已经保证了合法
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

所以添加和删除的实现也是最平常能够想到的，当我们要在某个位置插入或者删除元素时，必然要给它腾出位置来，我们现在关注一下扩展容量的函数。

#### 容量扩充

首先对于一个初始化的空的类型而言，它默认有的容量是10个单位。

```
	private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code  // 以下如果够用了就不用再grow了
        if (minCapacity - elementData.length > 0)  // 当前的实例长度并不能满足最小容量的需求
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {  //越是频繁的使用，增加的量越是多
        // overflow-conscious code
        int oldCapacity = elementData.length;  // 老的长度
        int newCapacity = oldCapacity + (oldCapacity >> 1);  // 新的长度是原来长度的3/2
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)  // 看是否超过最大限制
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow  // 如果加的数字使得minCapacity溢出了int的类型
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?  
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

那么属于这一系列的函数有很多，包括set，addAll， removeAll，clear，removeRange...

### 余下一些杂项

#### sort

直接引用了Array类里面的sort方法，简单明了

```
	@Override
    @SuppressWarnings("unchecked")
    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

#### clone

覆写了object类中的clone方法，是list的一个实现浅拷贝的方法

```
	public Object clone() {  //浅拷贝，覆盖父类的clone函数
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

---

上面是ArrayList中比较核心的一些函数的实现了，但是要发现在ArrayList这个文件中其实不只是实现了这些函数，还有一些其他的应用被存放在了封装的私有类中，其中包括遍历器和子列表的实现。

### 迭代器的实现

迭代器这个工具本身有助于我们遍历整个列表对象，我们在java中的使用如下：

```
Iterator<Integer> m = a.iterator();
```

在后面再来看具体这个Iterator这个类究竟是怎么实现的，当下主要看一下在ArrayList中迭代器如何实现。首先在这个文件中实现了一个私有类Itr实现了接口Iterator， 而我们所使用到的iterator方法就是实例化了一个itr对象。

在迭代器中最常用的方法是next方法：

```
	public E next() {
        checkForComodification();
        int i = cursor;  // cursor指下一个元素
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;  
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;  // cursor自动移到下一个位置
        return (E) elementData[lastRet = i];  
        // lastRet指的是上一个位置，并且函数返回了list中的元素
    }
```

当然我们可以跳过另一个常用的函数hasnext，这个函数就是比较当前的cursor是否已经到达了size长度了。

来看一下对应另一个实现的函数remove：

```
	public void remove() {
        if (lastRet < 0)  // 没东西删除
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet); // 调用arraylist中内部的一个remove函数
            cursor = lastRet; // 然后让cursor变为上一次的的位置
            lastRet = -1;  // 同时清空了lastRet的内容，意思是此处的删除只能删除本元素
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
```

可以说这个迭代器的实现就是借助了arraylist中的方法与元素实现的。其中除了itr迭代器类，还实现了一个ListItr的子类继承了Itr类。它相当于在Itr的基础上封装了一层，使得迭代器更加灵活。

在这个子类中新添加了一些控制cursor和获取位置的方法，比如在初始化的时候可以通过系数index来指定初始cursor的位置，同时通过hasprevious，nextIndex，previousIndex获取当前cursor的位置状态。子类中还添加了previous方法（参照next方法的实现），还有就是add和set（arraylist中的add和set）。

```
	public void add(E e) {
        checkForComodification();  
        // 这一步就是在看是否有重复操作了这个add方法，这是保证了这个操作只能发生在遍历器内部，如果在遍历过程中

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();  
        }
    }
```

---

# Vector

我为什么要把vector放在这里，而不另外开一个新的文件来放呢？仔细读一下vector这个文件，可以发现成员变量和成员方法很多都和arraylist几乎是一模一样。

Vector本身也是用数组实现的，对于这个数组而言也有着可以扩容的方法，**比ArrayList多的一点在于它还可以限制长度，函数setSize**

```
	public synchronized void setSize(int newSize) {
        modCount++;
        if (newSize > elementCount) {  
            ensureCapacityHelper(newSize);  // 如果新的长度超过了老长度，那么根据要求可以扩充数组容量
        } else {
            for (int i = newSize ; i < elementCount ; i++) {
                elementData[i] = null;  // 如果说设置的长度比老长度要短，那么此时就需要缩减长度，而丢弃多余的元素
            }
        }
        elementCount = newSize;
    }
```

那么其他的操作无非就是向数组中增加，删除，修改，查找。如果看一下代码会发现方法和arraylist使用的方法是一致的。那么问题就在于，为什么要在库中两个类似的结构呢？因为仔细看一下发现，vector和arraylist最大的不同在于，vector是线程安全的。所以在多线程中操作数据使用vector更加好，但是这样也牺牲了性能。由于vector中的方法基本都是同步的，因此调用方法时开销比arraylist大很多。而不考虑同步的角度，arraylist的速度会比vector快上很多，这就是为什么要在类库中实现两个不同的结构。

---

## 总结

1、 ArrayList有哪些优势？

​	那么ArrayList的本质就是数组，我们可以思考使用数组做查询有什么优势：它根据下标遍历和查找元素效率很高，并且arraylist可以自动扩容，对用户来说是透明的。并且存储元素的时候空间相对节省，只使用了一个数组元素。

2、ArrayList有哪些缺点呢？

​	ArrayList在增加元素和删除元素时都需要整体搬动后续数组元素，这样就有了额外开销。并且因为数组必须要给它一个初始的容量，所以每一次的插入都需要考虑是否扩容。

3、ArrayList的扩容

​	ArrayList扩容是原地扩容，使用了Array.copyOf，每次扩容是原来的1.5倍。

