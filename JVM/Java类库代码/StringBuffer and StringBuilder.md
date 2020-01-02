# StringBuffer and StringBuilder

这是两个处理字符串的工具类，我可以说，这两个工具类的实现应该是一模一样的。StringBuffer是线程安全的而StringBuilder是线程不安全的，但是它们两个的本意都是想让String字符串的内容可以变化而不新建一个对象。

这两个工具类的实现都继承了AbstractStringBuilder，并且真正方法的实现也都在这个类中，所以在这里就直接看这个类里的实现了，我们只要知道在上层的两个工具类就是调用了父类的相应的方法。

在这个类里一共两个成员变量：

* char value[]：存放字符串的数组
* int count：字符串的长度

这个类的初始化没什么好讲的，因为就是使用一个参数capacity，来初始化这个value数组，那么如果在比如StringBuffer中要用比如某个String进行初始化呢？则会增加str.length()+16的长度然后把字符串添加上去。

```
public StringBuffer(String str) {
    super(str.length() + 16);
    append(str);
}
```

## 添加元素

```
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull(); // 添加一个空，这个有点骚
    int len = str.length();
    ensureCapacityInternal(count + len); 
    // 这里又要涉及数组的扩容，长度每次都扩充成原来的两倍
    str.getChars(0, len, value, count);
    // 将str的0到len长度的字符串复制到value数组的count位置之后
    count += len;
    return this;
}

private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l'; // 就是插入了null字符串
    count = c;
    return this;
}

// 起始下面的append你可以看到的是，都是基本上就是调用原本包装类的getchars，或者像浮点数来说就是使用它的appendTo，我们在对应的包装类里看
```

## 删除元素

```
public AbstractStringBuilder delete(int start, int end) { // 删除从start到end之间的子串
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        end = count;
    if (start > end)
        throw new StringIndexOutOfBoundsException();
    int len = end - start;
    if (len > 0) {
        System.arraycopy(value, start+len, value, start, count-end);
        // 删除的方法就是将删除后的后面的子串复制到前面来
        count -= len;
    }
    return this;
}

public AbstractStringBuilder deleteCharAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    // 同样删除某个位置的一个字符也需要将后面的字符往前搬移
    System.arraycopy(value, index+1, value, index, count-index-1);
    count--;
    return this;
}
```

## 插入字符串

```
public AbstractStringBuilder insert(int offset, String str) {
    if ((offset < 0) || (offset > length()))
        throw new StringIndexOutOfBoundsException(offset);
    if (str == null)
        str = "null";
    int len = str.length();
    ensureCapacityInternal(count + len);
    // 先要给插入的位置先腾出一个位置，所以假设str的len，那么就需要将offset后面的字符串搬移到offset+len开始后的位置
    System.arraycopy(value, offset, value, offset + len, count - offset);
    // 然后给这个位置将str复制进去
    str.getChars(value, offset);
    count += len;
    return this;
}

// 下面的insert函数对于不同的类型，都是使用了valueOf转换成了string然后使用上面的insert函数插入了进去，除了单独插入char类型之外
```

## 杂项

charAt就不用多说了，只是返回了在系数index处的那个字符。那么很同理，setCharAt也一样，就是根据下标index去修改某个元素。

indexOf查询操作也没什么好说的，就是调用了string类型中的indexOf方法。

```
public void setLength(int newLength) {
    if (newLength < 0)
        throw new StringIndexOutOfBoundsException(newLength);
    ensureCapacityInternal(newLength);

    if (count < newLength) {
    	// 怎么说呢，应该就是从新长度开始，都给他赋\0
        Arrays.fill(value, count, newLength, '\0');
    }

    count = newLength;
}
```

### 替换函数

```
public AbstractStringBuilder replace(int start, int end, String str) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (start > count)
        throw new StringIndexOutOfBoundsException("start > length()");
    if (start > end)
        throw new StringIndexOutOfBoundsException("start > end");

    if (end > count)
        end = count;
    int len = str.length();
    int newCount = count + len - (end - start);
    ensureCapacityInternal(newCount); // 因为替换的过程中并不是直接的修改某些字符，而有可能导致字符串长度发生变化
	// 下面这两步就和之前的那个insert所做的一样了
    System.arraycopy(value, end, value, start + len, count - end);
    str.getChars(value, start);
    count = newCount;
    return this;
}
```

### 取子串

```
public String substring(int start, int end) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        throw new StringIndexOutOfBoundsException(end);
    if (start > end)
        throw new StringIndexOutOfBoundsException(end - start);
    return new String(value, start, end - start); // 返回一个新的string类型对象
}
```

### 逆置

```
public AbstractStringBuilder reverse() {
    boolean hasSurrogates = false;
    int n = count - 1;
    for (int j = (n-1) >> 1; j >= 0; j--) { // 从尾部开始找
        int k = n - j; // 头部位置向后
        char cj = value[j];
        char ck = value[k];
        value[j] = ck; // 交换元素
        value[k] = cj;
        if (Character.isSurrogate(cj) ||
            Character.isSurrogate(ck)) {
            hasSurrogates = true;
        }
    }
    if (hasSurrogates) {  // 这个应该是用来处理非常规类型的一些字符
        reverseAllValidSurrogatePairs();
    }
    return this;
}

private void reverseAllValidSurrogatePairs() {
    for (int i = 0; i < count - 1; i++) {
        char c2 = value[i];
        if (Character.isLowSurrogate(c2)) {
            char c1 = value[i + 1];
            if (Character.isHighSurrogate(c1)) {
                value[i++] = c1;
                value[i] = c2;
            }
        }
    }
}
```

---

## 总结

1、String，StringBuffer和StringBuilder哪一个是线程安全的？ StringBuffer

