# Java线程和线程安全

## 线程

当前的计算机为了解决处理器和内存之间的读写速度的缓和问题，就需要在处理器和内存之间加入一个缓存来协调，但是这样也引入了一个缓存一致性的问题。比如当在多处理器上，每个处理器都会有各自的缓存，每个缓存又共享了同一片内存空间的时候，如果同时写入数据到相同的地址，势必会带来读写一致性问题，这时只能根据相应的协议来执行操作。

### java内存模型

java内存模型的主要目标是定义程序中各个变量的访问规则，即实例字段、静态变量和数组元素，不包括线程私有的变量。这些变量都会存放在主内存中（虚拟机内存的一部分）。每条线程都有它自己的工作内存，这里面存放了在主内存中的变量的一个副本。那么某个线程读写变量都是在工作内存中完成。

虚拟机中对于变量的使用定义了8中原子操作，lock（独占某个变量）、unlock（释放锁定的变量）、read（主内存中读到工作内存中）、load（把read中从主内存中读到的变量放入工作内存的变量副本中）、use（将工作内存中的某个变量的值传递给执行引擎）、assign（把一个从执行引擎中接收到的值赋值给工作内存变量）、store（把工作内存变量中的值传送到主内存）、write（将工作内存中得到的变量的值放入主内存的变量中）。

volatile关键字是java虚拟机提供的轻量级同步机制，当一个变量定义了volatile后，它就保证了它对所有线程的可见性，意思是说当某个线程修改了值，其他线程会立刻知道。但是这不代表volatile变量的运算在并发下是安全的。使用volatile关键字一定要满足下面两个条件：运算结果不依赖当前的变量值，或者确保只有单一线程修改、变量不需要和其他的状态变量共同参与不变约束。并且volatile保证禁止了指令的重排序优化。

**一个有趣的可能是对于long和double这种需要读取两次32位数据的情况可能出现非原子性操作的可能，那么此时就可能出现读取修改了一半的数值，但是普遍的虚拟机中还是不会出现这种情况，因为在设计时还是会提倡将它们设定为原子操作。**

**在java中存在了先行发生规则，这是一个很好地判断线程并发执行是否存在安全问题的一个条件。在这个规则中，你既不能保证时间上的先发生代表这个操作会先执行，也不能保证操作先执行一定是执行时间先发生。**

### Java线程

在java中，一个已经执行了start但是还未结束的Thread类就是线程，在这个类中大部分的方法都使用了native来声明，这是因为java线程的实现是和硬件和操作系统平台相关，所以无法实现为平台无关。实现线程通常有如下方法：

* 内核线程：也就是内核中有一个调度器，可以分配线程到某个处理器上执行。为了让程序使用内核线程，在用户层面会实现一个高级接口——轻量级进程。而一个轻量级进程成为了一个独立的调度单元，但是可惜的是这种方法需要频繁调用系统调用，开销比较大。
* 用户线程：这类线程不需要内核的参与，全程由用户空间思考线程的创建、同步等操作。但是缺点就是这对于用户程序要求很高，缺乏了内核的支持。
* 混合模式：就是用户线程和内核线程都是用。

线程调度方面java使用了协同式调度（直到线程执行结束）和抢占式调度两种调度方法。

线程状态保证了每个线程在一个时间点只能处于一个状态下，这些状态包括了新建、运行、无限期等待（等待其他线程显示唤醒）、有限期等待（一定时间内由系统自动唤醒）、阻塞（等待获取一个排他锁）、结束

---

## 线程安全

线程安全的代码必须包含一个特征：代码本身封装了所有必要的正确性保障手段，让调用者无须关心多线程的问题，页无须自己采取任何措施来保证多线程的正确调用。

在java中按照线程安全的由强到弱，总共可以将操作共享数据的类型分为下面5类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

* 不可变的变量一定是线程安全的。对于普通的数据类型，加上final关键字保证了其不可变的性质，对于一个对象只要保证它的行为不会对本身的状态产生影响，比如String，这种情况下可以考虑将对象中的成员定义为final。
* 绝对线程安全满足由Brian Goetz提出的线程安全的定义，基本上JAVA API中使用的类都不是线程安全的。
* 相对线程安全就是通常意义上的线程安全，就是保证了对这个对象单独操作是线程安全的。但是在调用端需要额外的手段来保证调用的正确性。
* 线程兼容指对象本身不是线程安全的，但是要通过调用端正确使用同步手段来保证对象在并发环境下安全使用。
* 线程独立无法在多线程的环境中并发使用的代码。

### 实现

##### 互斥同步

在java中最基本的互斥同步手段是synchronized关键字，这个关键字背后的本质是将一个对象进行锁定和解锁的一个过程。但是synchronized是java中一个重量级的操作，它的意思会阻塞后续其他线程的进入。当然还可以使用concurrent中的重入锁来实现同步。

##### 非阻塞同步

互斥同步最大的问题在于阻塞和唤醒带来的性能问题，这种方式称为阻塞同步，并且属于一种悲观的处理方式（意思是说如果不进行这样的处理就肯定会出问题）。而非阻塞同步就是一种乐观的并发策略，先进性尝试，如果出现问题再进行弥补。这里需要硬件来帮忙操作。

##### 无同步

有一些代码天生就是线程安全的，比如**可重入代码**，意思就是在执行时打断代码执行后依旧能够正确地执行原代码；**线程本地存储**，意思是虽然代码中使用了共享数据，但是要看是否保证这些代码在一个线程中执行。

### 锁优化

* 自旋锁：让线程执行一个忙循环。如果锁被占用的时间很短，效果就很好；如果占用时间很长，那么自旋就会消耗处理器资源，因为自旋本身需要占用处理器时间。从1.6版本以后自旋锁变成了自适应的，查看申请锁的线程是否可能拿到锁。
* 锁消除：如果在执行的代码中检测到并不可能存在共享数据竞争的情况，则可以将锁消除。
* 锁粗化：如果一系列连续的操作都是对同一个对象反复加锁和解锁，那么可以进行粗化锁的操作，就是在整个操作的开始处和结束处加锁。
* 轻量级锁：这个和对象中的Mark Word有关，其中有两位专门存放的是锁的标志位，此时如果需要上锁，那么就会现在栈帧中进阿里一个锁记录，再将这个锁记录复制到对象头，标志位置位，这就是获取了锁。如果有两条以上线程争用了同一个锁，则需要膨胀为重量级锁。解锁的过程就是逆向替换的过程。
* 偏向锁：在轻量级锁的基础上将锁的线程ID加入到对象的头结构，这就是偏向，随后如果锁定的话就仍然需要使用轻量级锁的方法。意思是这个锁永远是偏向于第一个申请它的线程，如果一直是这个线程使用，则不需要进行同步的操作。