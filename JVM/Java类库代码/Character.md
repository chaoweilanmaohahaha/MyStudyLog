# Character

为什么要单独看一下Character这个类，通过这个类，我们总结一下也回顾一下字符的操作，我们要着重看一下这里处理unicode(UTF-16)编码的方法，那么Java中怎么处理这些不同编码的字符呢？

首先在这个文件中，一些默认参数的定义是真的多，如果这里一一列举肯定爆炸，那这些所谓的默认参数就在用到的时候再说好了。那也就是直接来看这个类里的成员变量：

private final char value; 存放了这里面保存的字符

所以在初始化的时候也很简单：

```
public Character(char value) {
    this.value = value;
}
```

## 杂项

每个包装类里都有几个特定的函数，比如valueof，toString等，这里的valueOf参考以下Integer里所述，它同样给部分值设置了一个缓存：

```java
public static Character valueOf(char c) {
    if (c <= 127) { // must cache
        return CharacterCache.cache[(int)c];
    }
    return new Character(c);
}

private static class CharacterCache {
    private CharacterCache(){}

    static final Character cache[] = new Character[127 + 1];

    static { // 静态代码块，这里要学习以下，还不知道有什么不同的呢
        for (int i = 0; i < cache.length; i++)
            cache[i] = new Character((char)i);
    }
}
```

```java
public String toString() { // toString比较波澜不惊，就是创建了一个字符数组
    char buf[] = {value};
    return String.valueOf(buf);
} 
```

Character对于hashcode的操作也是将自己的(int)value值赋给自己，也没什么好说的。

## 一些检验

在这个文件中存在着很多的用来检验的函数，正好通过这些函数来看一些变量为什么这样设计：

```java
// 检测码值是否落在了unicode码值合法范围内
public static boolean isValidCodePoint(int codePoint) {
    // Optimized form of:
    //     codePoint >= MIN_CODE_POINT && codePoint <= MAX_CODE_POINT
    int plane = codePoint >>> 16;
    return plane < ((MAX_CODE_POINT + 1) >>> 16);
}

public static final int MIN_CODE_POINT = 0x00000； // unicode码最小值
public static final int MAX_CODE_POINT = 0X10FFFF; // unicode码最大值

public static final int MIN_SUPPLEMENTARY_CODE_POINT = 0x010000; 
// 上面这个值是代表了UTF-16中的辅助平面的起始，具体的看一下总结中的unicode码的说法吧
// 所以就有了下面判断是否落在了辅助平面内
public static boolean isSupplementaryCodePoint(int codePoint) {
    return codePoint >= MIN_SUPPLEMENTARY_CODE_POINT
        && codePoint <  MAX_CODE_POINT + 1;
}

// 0x0000 - 0xFFFF这个区间也就是UTF-16的第一个平面，称为平凡平面
public static boolean isBmpCodePoint(int codePoint) {
    return codePoint >>> 16 == 0;
}
```

```java
// 下面的两个参数限定了unicode真正代表了字符的一些编码阈值
public static final char MIN_HIGH_SURROGATE = '\uD800';
public static final char MAX_HIGH_SURROGATE = '\uDBFF';
public static final char MIN_LOW_SURROGATE  = '\uDC00';
public static final char MAX_LOW_SURROGATE  = '\uDFFF';
// 所以下面就相当于在判断是否是一个可以代表字符的unicode码
public static boolean isHighSurrogate(char ch) {
    // Help VM constant-fold; MAX_HIGH_SURROGATE + 1 == MIN_LOW_SURROGATE
    return ch >= MIN_HIGH_SURROGATE && ch < (MAX_HIGH_SURROGATE + 1);
}

public static boolean isLowSurrogate(char ch) {
    return ch >= MIN_LOW_SURROGATE && ch < (MAX_LOW_SURROGATE + 1);
}

public static boolean isSurrogate(char ch) {
    return ch >= MIN_SURROGATE && ch < (MAX_SURROGATE + 1);
}

public static boolean isSurrogatePair(char high, char low) {
    return isHighSurrogate(high) && isLowSurrogate(low);
} // 注意上面的参数中都是UTF-16字符
```

```java
// 检测字符个数，以辅助平面0x10000划分
public static int charCount(int codePoint) {
    return codePoint >= MIN_SUPPLEMENTARY_CODE_POINT ? 2 : 1;
} // 如果超过了辅助平面那就说明是一个辅助字符，字符长度是2。否则是1

public static int toCodePoint(char high, char low) { 
    // 这个应该就是将两个属于同一个字符的两个字节拼接起来，形成一个4个字节的字符
    return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT
                                   - (MIN_HIGH_SURROGATE << 10)
                                   - MIN_LOW_SURROGATE);
}

public static int codePointAt(CharSequence seq, int index) {
    // 这个函数中间获得一个字符串中的某个字符，方法就是要先判断前两个字节是否落在了一个空段中，如果落在空段中证明这是一个4字节字符
    char c1 = seq.charAt(index);
    if (isHighSurrogate(c1) && ++index < seq.length()) {
        char c2 = seq.charAt(index);
        if (isLowSurrogate(c2)) {
            return toCodePoint(c1, c2);
        }
    }
    return c1;
}

static int codePointAtImpl(char[] a, int index, int limit) {
    char c1 = a[index];
    if (isHighSurrogate(c1) && ++index < limit) {
        char c2 = a[index];
        if (isLowSurrogate(c2)) {
            return toCodePoint(c1, c2);
        }
    }
    return c1;
}
```

那么下面就围绕着这个字符是在普通的平面上还是在辅助平面上设计了许多函数

```java
static int codePointCountImpl(char[] a, int offset, int count) {
    int endIndex = offset + count;
    int n = count;
    for (int i = offset; i < endIndex; ) {
        if (isHighSurrogate(a[i++]) && i < endIndex &&
            isLowSurrogate(a[i])) {
            n--;
            i++;
        }
    }
    return n;
} // 字符计数，看a中到底有多少字符
```

下面是判断大小写：

```java
public static boolean isLowerCase(int codePoint) {
    return getType(codePoint) == Character.LOWERCASE_LETTER ||
           CharacterData.of(codePoint).isOtherLowercase(codePoint);
}

public static boolean isDigit(int codePoint) {
    return getType(codePoint) == Character.DECIMAL_DIGIT_NUMBER;
}

...
    
public static int getType(int codePoint) {
    return CharacterData.of(codePoint).getType(codePoint);
}
```



---

# 总结

1、UTF-16编码以及其他

​	我感觉要看懂这个文件，必须先了解以下unicode码。之所以使用这个码，是因为ascii码根本不可能用来表示所有的字符。为了表示世界上几乎所有的字符，扩充字符长度成了最简单的一个方法。但是如果单纯的扩充长度势必会带来空间的浪费，所以提出了UTF-8和UTF-16字符的编码方式。

​	UTF-8编码实现了ASCII码的向后兼容，它的最大的好处就是可以变换长度。它的编码方式为：

* 假设当个字节的字符，则在UTF-8中需要将第一位设置为0，其余对应unicode码点。也就是：

  0000 0000 - 0000 007F --- >  0xxxxxxx

* 那么如果有n个字符，那么就需要让第一个字节为1，则让第一个字节的前N位为1，剩余N-1个字节前两位都设为10，然后所有x位置用来填充unicode码点，也就是：

  0000 0080 - 0000 07FF ---> 110xxxxx 10xxxxxx

这样相当于将unicode化成了标准的UTF-8编码。那么为了让unicode码可以包含几乎所有的字符，那么设计者就设计出了一种技术，它将unicode码进行分区，每个区让他存放65536个字符，而把这样一个区给个名字叫做平面。那么在unicode码中一共有17个这样的平面。

最前面的65536个字符称为基本平面，也就是BMP（U+0000 到 U+FFFF），而剩下的字符都在辅助平面内，就是SMP（U+010000 到 U+10FFFF）。那么可以看到这里有个问题，如果我们遇到了两个字节，到底它属于一个字符还是一个辅助平面呢？

在unicode中使用了一个很巧妙的方法，在基本平面内，设置了一个空段它不映射任何的字符（U+D800 到 U+DFFF ）。而一个辅助平面的字符占到2的20次方，至少需要20位，那么在UTF-16编码中，将这个20位一切为2，前10为映射到上面那个空段中的U+D800 到 U+DBFF，这个称为高位（HighSurrgorate），后面10位映射到U+DC00 到 U+DFFF，这个称为低位（LowSurrgorate）。所以判断的标准在于，如果你发现两个字节的码点落在了U+D800 到 U+DBFF中时，那么可以断言后面还跟着两个字节，这四个字节需要组合在一起读。