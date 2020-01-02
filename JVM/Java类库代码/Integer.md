# Integer

这是接触的第一个包装类，整数类。包装类主要要看一下java中的所谓装箱和卸箱的做法。当然本身Integer的做法也是要学习的。

Integer是一个有符号整数类所以它的数值范围达到了-2^31 ~ 2^31-1。这个阈值是在类里显式定义的：

```
public static final int   MIN_VALUE = 0x80000000;
public static final int   MAX_VALUE = 0x7fffffff;
```

说句实话，这个类写的有点乱，看了半天很难找到这个成员变量在哪里，那么在这个类里指维护了一个成员变量：value用来存放当前的值，那么初始化这个Integer的操作有点顺利成章的。

## 初始化

```java
public Integer(int value) {
    this.value = value;
}

public Integer(String s) throws NumberFormatException { // 会抛出格式异常错误
    this.value = parseInt(s, 10);
}

public static int parseInt(String s, int radix) throws NumberFormatException
{
    /*
     * WARNING: This method may be invoked early during VM initialization
     * before IntegerCache is initialized. Care must be taken to not use
     * the valueOf method.
     */
	// 上面的注释解释了为什么不能使用valueOf这个方法
    if (s == null) {
        throw new NumberFormatException("null");
    }

    if (radix < Character.MIN_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " less than Character.MIN_RADIX");
    }

    if (radix > Character.MAX_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " greater than Character.MAX_RADIX");
    }
	// 上面在判断输入的合法性
    int result = 0;
    boolean negative = false;
    int i = 0, len = s.length();
    int limit = -Integer.MAX_VALUE;
    int multmin;
    int digit;

    if (len > 0) {
        char firstChar = s.charAt(0); // 先取出第一个字符因为可能第一个字符是'+'和'-'。
        if (firstChar < '0') { // Possible leading "+" or "-"
            if (firstChar == '-') { // 如果是负数，就需要置位negative标志
                negative = true;
                limit = Integer.MIN_VALUE;
            } else if (firstChar != '+') // 如果不是'+'和'-'则输入不合法
                throw NumberFormatException.forInputString(s);

            if (len == 1) // Cannot have lone "+" or "-"
                throw NumberFormatException.forInputString(s);
            i++;
        }
        multmin = limit / radix;
        while (i < len) {
            // Accumulating negatively avoids surprises near MAX_VALUE
            digit = Character.digit(s.charAt(i++),radix); // 计算这个基数下的值
            if (digit < 0) {
                throw NumberFormatException.forInputString(s);
            }
            if (result < multmin) {
                throw NumberFormatException.forInputString(s);
            }
            result *= radix;
            if (result < limit + digit) {
                throw NumberFormatException.forInputString(s);
            }
            result -= digit;
        }
    } else {
        throw NumberFormatException.forInputString(s);
    }
    return negative ? result : -result;
} // 这里使用了一个技巧，它为了防止生成的数超过最大或者最小值，所以它选择让数字逆向增长，这就是为什么让limit设置为-Integer.MAX_VALUE。very巧妙！
// 当然其中还有一个parseUnsignedInt函数，用来将字符串转换成整数
```

## 转换函数

上面的parseInt其实就是将字符串转换为了整数，那么在Integer类里，充满了从整数转换到各种类型以及从各种类型转换成整数的方法。

### 整数转换为其他类型

先来看一下转换为字符串

```java
public static String toString(int i, int radix) {
    if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
        radix = 10;

    /* Use the faster version */
    if (radix == 10) {  // 如果基数为10，则就调用我们最常用的那个转换函数
        return toString(i);
    }

    char buf[] = new char[33];
    boolean negative = (i < 0);
    int charPos = 32;

    if (!negative) { // 如果是负数，那就先转换为正整数
        i = -i;
    }

    while (i <= -radix) { // 从后往前插入到预留的数组中
        buf[charPos--] = digits[-(i % radix)]; 
        // 这里也很巧妙，为什么要模个负的呢？因为我现在想取的数其实是0-radix，那么这样使用负数取模正好就是这个区间而不用改变radix
        i = i / radix;
    }
    buf[charPos] = digits[-i];

    if (negative) {
        buf[--charPos] = '-';
    }

    return new String(buf, charPos, (33 - charPos)); // 返回一个字符串对象
}

public static String toString(int i) {
    if (i == Integer.MIN_VALUE)
        return "-2147483648";
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i); 
    // stringsize就是根据文件中给定的size表来设置长度：
	/*
		final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
                                      99999999, 999999999, Integer.MAX_VALUE };
	*/
    char[] buf = new char[size];
    getChars(i, size, buf);
    return new String(buf, true); // 返回一个新的字符串对象
}

// 稍微看一下getChars函数
static void getChars(int i, int index, char[] buf) { // 将这个整数转换成字符数组
    int q, r;
    int charPos = index;
    char sign = 0;

    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // Generate two digits per iteration
    // 下面据说又是使用了一个技巧
    while (i >= 65536) { // 如果数字足够大的化，就采用快速神变成数据的方法
        q = i / 100;
    // really: r = i - (q * 100);
        r = i - ((q << 6) + (q << 5) + (q << 2)); // ？？？ 666 哦
        i = q;
        buf [--charPos] = DigitOnes[r]; // 查表可得个位数上的值
        buf [--charPos] = DigitTens[r]; // 查表可得十位数上的值
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i <= 65536, i);
    for (;;) {
        q = (i * 52429) >>> (16+3); // i * 0.1 //这也太秀了吧
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
        buf [--charPos] = sign;
    }
}
```

### 整数转换成多进制

转换进制本质使用了函数toUnsignedString0

```java
private static String toUnsignedString0(int val, int shift) {
    // assert shift > 0 && shift <=5 : "Illegal shift value";
    int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);
    char[] buf = new char[chars];

    formatUnsignedInt(val, shift, buf, 0, chars);

    // Use special constructor which takes over "buf".
    return new String(buf, true);
}

static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
    int charPos = len;
    int radix = 1 << shift; // 根据偏移计算基数，比如shift为4则radix为16
    int mask = radix - 1;
    do {
        buf[offset + --charPos] = Integer.digits[val & mask]; // 通过掩码来确定插入的值，插入的位置由offset确定
        val >>>= shift; // 这个整数移位，就是除了这么多
    } while (val != 0 && charPos > 0);

    return charPos;
}
```

### 整数还有转换成其他数据类型

类似byte，short，long，float，double，就只要使用强制类型转换就可以了

### 其他类型转换成整数

转换成整数的方法的一种是使用了valueOf方法

```java
public static Integer valueOf(String s) throws NumberFormatException {
    return Integer.valueOf(parseInt(s, 10));
}

public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high) 
        // 这个所谓的IntegerCache就是一个-128~127的一个缓存，如果这个数据在这个里面就可以直接拿到
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

## 杂项

对于整形的hashcode计算，就是返回了整数的值自身。

比较函数呢，这里可以发现：

```java
public static int compare(int x, int y) {
    return (x < y) ? -1 : ((x == y) ? 0 : 1); // 如果前者比后者小返回-1，相等返回0，否则返回1
}

public boolean equals(Object obj) {
    if (obj instanceof Integer) { // 同类型进行比较
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

其他就实现了整数的一些基本的运算函数，其中包括：

* max：取最大值；
* min：取最小值；
* sum：取和；
* reverse：逆置；
* rotateLeft，rotateRight：循环左移和循环右移
* highestOneBit， lowestOneBit：取最高位和最低位

---

## 总结

其实这个包装类中的方法是实现了一些整数上的方法，但是讲到包装类其实更重要的问题是Java中所包含的自动装箱和自动拆箱的机制，这个机制是具体怎么实现的呢？

在java中有包括int，boolean这样的基本数据类型，所谓基本数据类型，其实就是指这个数据在内存中的存放格式，比如int是4字节，double是8字节。那么每一种数据类型会对应一个它的包装类，就像Integer那样，里面封装了许多的操作方法。那么看一下以下代码，这是比较常见的一个使用：

```
Integer i = 1; // 装箱
Integer i = new Integer(1);
```

那么如果我们运行到了第一句话，那么它本质是在运行Integer的valueOf方法，这就是一个装箱，但是这里就有个细节了。我们刚才看到了一个IntegerCache结构，所以在装箱的过程中还有区别，我们以下面的代码来看：

```
Integer a = 123;
Integer b = 123;
System.out.println(a == b); // true
// 如果是这样装箱，那么在java虚拟机会对-127~128的数建立一个cache，那么如果我们取这里面的数就会直接取cache中拿，那么也就是说，a和b会指向同一个对象。


Integer c = 129;
Integer d = 129;
System.out.println(c == d); // false
// 超过128的数值就需要新建一个Integer对象，这个对象就开辟在堆中，所以它们指向不同的对象
```

那么如果反过来我们这样定义呢？

```
Integer i = 1;
int g = i; // 拆箱
```

这就是拆箱过程，那么如果我们执行到第二句，其实就是在执行类中的intValue方法，返回的就是value值，所以我们看到下面的比较

```
int e = 129;
Integer f = new Integer(129);
System.out.println(f == e); // true
// 拆箱后，这是值之间的比较
```

