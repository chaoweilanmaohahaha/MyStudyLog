# redis 对象

基于 redis 中的数据结构，redis 构造了自己的对象系统。系统中一共包括了五种类型的对象，分别是字符串对象、列表对象、哈希对象、集合对象和有序集合对象。

一个 redis 对象由一个 redisObject 结构表示：

```
typedef struct redisObject {
	unsigned type;
	unsighed encoding;
	void *ptr;
	...
}
```

redis 数据库保存的是键值对，键总是一个字符串对象，而值可以是任意一种类型的对象。这里的 type 标识了它是五种类型中的哪一种。为什么需要 encoding 这个标志，是因为 redis 中的每一种对象都有不止一种底层实现(为了优化)，为了给出当前这个对象是用了哪一种实现，需要 encoding 这个数值来表明。

## 字符串对象

对于字符串而言有三种可能的编码：int、raw 和 embstr。

如果该对象保存的是一个整数值，范围在 long 内，则 ptr 直接指向一个整数，编码置为 int；

如果该对象保存的是一个字符串值，并且长度大于 39 个字节则使用一个 SDS 来保存，并将编码置为 raw；

如果该对象保存的是一个字符串值，并且长度小于 39 个字节则使用 embstr 编码保存(只读)，这种编码的方法就是使用一次内存分配函数直接分配一个 redisObject + sdshdr。

所以字符串对象在 redis 中能够保存整数，浮点数以及字符串。

## 列表对象

列表对象的编码可以是 ziplist 和 linkedlist(*注意的是链表中的每个节点指向的也是一个对象*)。

## 哈希对象

哈希对象的编码可以是 ziplist 和 hashtable。使用压缩列表的时候，保存同一个键值对的节点永远是紧挨在一起的，键在前值在后。

## 集合对象

集合对象的编码可以是 intset 和 hashtable。很显然如果在这个集合中都保存的是整数并且满足一定条件时，就使用 intset 保存。

## 有序集合对象

有序集合对象可以是 ziplist 和 skiplist。如果使用 ziplist，则每个元素使用两个紧挨在一起的节点表示，前者表示元素的成员，后者表示元素的 score。

这里的 skiplist 并不是单纯的跳跃表，而是使用了一种 zset 结构来实现：

```
typedef struct zset {
	zskiplist *zsl;
	dict *dict;
}
```

也就是说 skiplist 同时使用了跳跃表和字典来实现，首先它在具体实现的时候并不会浪费额外的内存。**同时使用两个对象的原因主要是针对查找和范围型操作都可以很快地执行**。

----

## 对象操作

redis 中操作键的命令基本上分为两种类型，其中的一种可以对任何类型的键执行，比如说 DEL，EXPIRE 等；而另一种键只能对特定类型执行。针对特定类型的命令，在进行时必须先进行类型检查，这是通过 redisObject 结构中的 type 属性实现的。当然针对 type 来选择处理不同对象的操作，还需要针对不同的 encoding 来选择不同的底层操作，这个在 redis 中可以认为某些命令是**多态**的。

C 语言不像 Java 那样具有自动内存回收功能，所以在 redisObject 中有一个 refcount 属性，这是该对象的引用计数。当这个引用计数一旦变为 0 时，对象所占用的内存会被释放。有了引用计数，redis 还能够实现对象共享，也就是说当两个键同时创建了保存相同内容的值对象时，两个键就可以共享这个值对象(好像这样的共享对象只有保存了整数值的对象)。

redisObject 还包含了 lru 属性，记录了某个对象的**空转时长**，这是对象最后一次被命令程序访问的时间。如果服务器占用的内存已经超过了 maxmemory 选项所设置的上限值时，空转时长较大的那部分键就会被优先释放。

