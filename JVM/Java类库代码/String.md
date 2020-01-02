# String

这里看一下String的实现，为什么还要看String的实现呢，主要是为了后面看StringBuffer和StringBuilder这两个类。

## 说在前面

对于java中的字符串而言，字符串常量都是用这个类实现的。并且String一经被定义出来，它们的值就不能被更改了，额外的修改就是引用另一个String了。

比如String a = "abc", 并且我们如果直接打"abc"依然可以使用String中的方法。那么虚拟机怎么知道字符串常量是一个对象的呢？**这个问题暂时先保留，忙猜可能虚拟机内建立了一个匿名对象**

String包含的成员变量有：

* final char value[]： 底层还是用字符数组来存储字符串；
* int hash： 存储String 的hashcode

## 初始化

最简单的，我们在创建一个空字符串的时候要做的

```java
public String() {
    this.value = "".value;  // 所以new String() 相当于让变量 = ""
}

public String(String original) { 
    this.value = original.value;
    this.hash = original.hash;
} // 用一个已有的String对象来创建一个新的String对象，这里可以看到这样初始化String的本质了
// 创建一个新的String对象，是让其中的value变量指向了原来那么String对象的value处，但是本身是一个新的对象

public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
} // 用数组进行初始化

public String(char value[], int offset, int count) // 使用一个数组进行初始化，并且只挑选其中部分元素。

public String(int[] codePoints, int offset, int count) // 使用unicode码数组来初始化字符串，这个难点是对unicode码的处理后需要计算到底需要用到多大的value数组的容量，这是需要通过codePoint数组去计算出来的。
// 当然 String还可以使用8bits的byte类型数组来初始化，这里就不再叙述了

public String(StringBuffer buffer) {
    synchronized(buffer) {  // 同步
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
} // 使用了StringBUffer来初始化了字符串，这里使用了同步的代码块

public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}
```

## 常用的方法

下面逐一来看一下String中常用到的那些方法到底是怎么实现的呢？

首先length和empty方法就不用多说了，就是判断value数组的长度。

```java
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {  // 判断一下index的合法性
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```

关于unicode编码的问题，其中有一个函数簇：codePointAt，codePointBefore，codePointCount，这个等到看到Character的时候再回过头来看一下。

### get函数簇

这个函数簇更像是在将String复制到某个数组中，比如常用的getChars。这是两个不同的数组。

```java
void getChars(char dst[], int dstBegin) {
	// 函数中不做任何的合法性检测
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}

public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {  // 代码中常用的
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

或者就是getBytes

```java
public void getBytes(int srcBegin, int srcEnd, byte dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    Objects.requireNonNull(dst);

    int j = dstBegin;
    int n = srcEnd;
    int i = srcBegin;
    char[] val = value;   /* avoid getfield opcode */  // 浅拷贝一份

    while (i < n) {
        dst[j++] = (byte)val[i++];
    }
}
```

### 比较

这里包含了最常用的两个比较函数，equals比较两个字符串值是否相等，这个不同于使用==号哦！还有就是一个compareTo，比较两个字符串谁更大：

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) { // 如果这确实是一个String对象
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {  // 先判断长度是否相同
            char v1[] = value; // 使用一个引用，这样做有什么好处吗？
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) { // 循环逐个比较
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}

public boolean equalsIgnoreCase(String anotherString) { // 忽略大小写的一个比较
    return (this == anotherString) ? true
            : (anotherString != null)
            && (anotherString.value.length == value.length)
            && regionMatches(true, 0, anotherString, 0, value.length);
}
```

看一下附带的regionMatches函数的实现：

```java
public boolean regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int len) { // 其实这个函数还有一个版本，就是直接看某个区域内的元素是否相等
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }  // 以上均是合法性验证，看下面才是重头戏
    while (len-- > 0) {
        char c1 = ta[to++];
        char c2 = pa[po++];
        if (c1 == c2) {
            continue;
        }
        if (ignoreCase) { // 如果要看大小写，则比较的方法是将当前的两个比较对象同时大写并且同时小写都比较一次，如果这个ignoreCase标志设置为true，就直接忽略下面这一段代码了
            char u1 = Character.toUpperCase(c1);
            char u2 = Character.toUpperCase(c2);
            if (u1 == u2) {
                continue;
            }
            if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                continue;
            }
        }
        return false;
    }
    return true;
}
```

```java
public int compareTo(String anotherString) { // 比较大小，最后有几种用来比较的指标
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2); 
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {  // 可以看到，就算是字符串长度不等，首先看的指标仍然是ascll码的大小 
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2; // 如果前缀大小一样，就返回字符串长度的区别
}
```

当然compareTo也有忽略掉大小写的一套方法，那么根据这类方法，就萌生出了一系列比较字符串大小的函数，包括startswith，endswith用来比较目标串的前缀和后缀是否相同的函数。

### hashcode

字符串类给出了它自己实现的计算hashcode的方法

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
           	// 方法就是 s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

### 查找函数

查找字符串的函数也是字符串中最常用的那些函数之一了。

```java
public int indexOf(int ch, int fromIndex) {  // 找某个字符哦
    final int max = value.length;
    if (fromIndex < 0) {
        fromIndex = 0;
    } else if (fromIndex >= max) {
        // Note: fromIndex might be near -1>>>1.
        return -1;
    }
	// 上面在做合法性验证，下面开始函数主体
    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) { // 后面这个标志的含义具体要看Character中的定义了
        // handle most cases here (ch is a BMP code point or a
        // negative value (invalid code point))
        final char[] value = this.value;
        for (int i = fromIndex; i < max; i++) { // 逐个进行查找
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return indexOfSupplementary(ch, fromIndex);  // 补充字符
    }
}

// 下面是找某个字符串的流程，有点绕，不过其实最后要绕到比较字符数组的内容
public int indexOf(String str) {
    return indexOf(str, 0);
}

public int indexOf(String str, int fromIndex) {
    return indexOf(value, 0, value.length,
            str.value, 0, str.value.length, fromIndex);
}

static int indexOf(char[] source, int sourceOffset, int sourceCount,
            char[] target, int targetOffset, int targetCount,
            int fromIndex) {
    if (fromIndex >= sourceCount) {
        return (targetCount == 0 ? sourceCount : -1);
    }
    if (fromIndex < 0) {
        fromIndex = 0;
    }
    if (targetCount == 0) {
        return fromIndex;
    }
    // 以上对于系数的合法性验证
    char first = target[targetOffset];
    int max = sourceOffset + (sourceCount - targetCount);

    for (int i = sourceOffset + fromIndex; i <= max; i++) {
        /* Look for first character. */
        if (source[i] != first) {  
            // 这下面的循环就是在找从开头开始第一个和子串的第一个字母相同的位置
            while (++i <= max && source[i] != first);
        }

        /* Found first character, now look at the rest of v2 */
        if (i <= max) {
            int j = i + 1; 
            int end = j + targetCount - 1;
            for (int k = targetOffset + 1; j < end && source[j]
                    == target[k]; j++, k++); 
            // 从刚才找到的位置开始，进行逐个比较，看是否有相同的串

            if (j == end) { // 如果走完了上面的循环，发现找到了一个相同的串，则直接返回子串在母串中第一个字母出现的位置
                /* Found whole string. */
                return i - sourceOffset;
            }
        }
    }
    return -1;
}
```

### 子串

```java
public String substring(int beginIndex, int endIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    if (endIndex > value.length) {
        throw new StringIndexOutOfBoundsException(endIndex);
    }
    int subLen = endIndex - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
            : new String(value, beginIndex, subLen);  
    // 从这边你就发现，如果想取的串不和本身相同，就会新建一个String对象
}
```

### 连接

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen); // 得到一个数组，是将value扩充了容量的数组
    str.getChars(buf, len); // 填充buf的扩充出来的区域
    return new String(buf, true); // 同样新建一个String对象
}
```

### 替换

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) { // 这里先计算一下需要交换的元素的第一个位置
        // 但是为什么不直接遍历一遍数组呢？
            if (val[i] == oldChar) {
                break;
            }
        }
        if (i < len) { // 如果确实需要替换
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}

```

在这里应该继续讲到**按照某个模式去替换以及将字符串按照某个模式去切割**，但是这中间都会用到正则表达式的操作，等学习了正则表达式的内容后再回到这里来。

### 去空

```java
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    while ((st < len) && (val[st] <= ' ')) { // 从头开始向后搜索，在这个比较中看到，并不是只有空格被去掉了，比空格小的字符也会被去掉
        st++;
    }
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```

最后看一下切换大小写的两个函数中的其中一个，因为功能实现十分类似，因此只需要看其中的一个就行了。

```java
// 我一开始也不相信， 这个函数这么长，但是仔细看看内容呢，但是还是等看完Character之后再回来看了，里面牵涉太多有关Character的内容了.
// 我回来了，起始就是在检测对于unicode码来说，码值是两个字节还是四个字节，但是根本上的uppercase还需要调用CharacterData里的方法，我佛了。
public String toUpperCase(Locale locale) {
    if (locale == null) {
        throw new NullPointerException();
    }

    int firstLower;
    final int len = value.length;

    /* Now check if there are any characters that need to be changed. */
    scan: { 
        for (firstLower = 0 ; firstLower < len; ) {
            int c = (int)value[firstLower];
            int srcCount;
            if ((c >= Character.MIN_HIGH_SURROGATE)
                    && (c <= Character.MAX_HIGH_SURROGATE)) {
                c = codePointAt(firstLower);
                srcCount = Character.charCount(c);
            } else {
                srcCount = 1;
            }
            int upperCaseChar = Character.toUpperCaseEx(c);
            if ((upperCaseChar == Character.ERROR)
                    || (c != upperCaseChar)) {
                break scan;
            }
            firstLower += srcCount;
        }
        return this;
    }

    /* result may grow, so i+resultOffset is the write location in result */
    int resultOffset = 0;
    char[] result = new char[len]; /* may grow */

    /* Just copy the first few upperCase characters. */
    System.arraycopy(value, 0, result, 0, firstLower);

    String lang = locale.getLanguage();
    boolean localeDependent =
            (lang == "tr" || lang == "az" || lang == "lt");
    char[] upperCharArray;
    int upperChar;
    int srcChar;
    int srcCount;
    for (int i = firstLower; i < len; i += srcCount) {
        srcChar = (int)value[i];
        if ((char)srcChar >= Character.MIN_HIGH_SURROGATE &&
            (char)srcChar <= Character.MAX_HIGH_SURROGATE) {
            srcChar = codePointAt(i);
            srcCount = Character.charCount(srcChar);
        } else {
            srcCount = 1;
        }
        if (localeDependent) {
            upperChar = ConditionalSpecialCasing.toUpperCaseEx(this, i, locale);
        } else {
            upperChar = Character.toUpperCaseEx(srcChar);
        }
        if ((upperChar == Character.ERROR)
                || (upperChar >= Character.MIN_SUPPLEMENTARY_CODE_POINT)) {
            if (upperChar == Character.ERROR) {
                if (localeDependent) {
                    upperCharArray =
                            ConditionalSpecialCasing.toUpperCaseCharArray(this, i, locale);
                } else {
                    upperCharArray = Character.toUpperCaseCharArray(srcChar);
                }
            } else if (srcCount == 2) {
                resultOffset += Character.toChars(upperChar, result, i + resultOffset) - srcCount;
                continue;
            } else {
                upperCharArray = Character.toChars(upperChar);
            }

            /* Grow result if needed */
            int mapLen = upperCharArray.length;
            if (mapLen > srcCount) {
                char[] result2 = new char[result.length + mapLen - srcCount];
                System.arraycopy(result, 0, result2, 0, i + resultOffset);
                result = result2;
            }
            for (int x = 0; x < mapLen; ++x) {
                result[i + resultOffset + x] = upperCharArray[x];
            }
            resultOffset += (mapLen - srcCount);
        } else {
            result[i + resultOffset] = (char)upperChar;
        }
    }
    return new String(result, 0, len + resultOffset);
}
```

---

## 总结

1、有关字符串的一些必要知识点

​	首先字符串在Java中充当的是一个常量，也就是说它本身的内容是不会改变的，生成另一个字符串的方法只有新产生一个字符串变量。字符串常量是保存在JVM虚拟机中运行时常量池中的，因此如果以一下的方式去定义：

```
String a = "abc";
String b = "abc";
a == b : true
```

为什么，因为a和b相当于同时直接指向了同一个常量池中的对象，那么这里的比较中==比较的是左右两边的对象是否为一个对象，此时就是相等的。那么我们看下面的代码：

```
String a = "abc";
String b = new String("abc");
String c = new String("abc");
a == b : false
b == c : false
```

因为相当于b和c中开辟了一个空间装载了String类对象，而这个对象中变量引用部分指向了常量池中同一个abc对象，而b和c本身并不是同一个对象。

