# 1.概述

Lucene通过**对象复用**和**内存分页**的思想，来优化java中GC的问题。Lucene设计了一系列高效的数据结构，其中就包括BytesRef。

BytesRef内部维护了一个字节数组的引用,以及一个偏移量和长度,读取数据时根据偏移量和长度进行截取。

在Lucene中很多地方都会用到数组，如ByteBlockPool中的二维数组buffers，其中每个元素buffer(block)就是一个一维的byte[]，索引过程中我们需要频繁读取block中存储的数据，如果进行数组的拷贝效率会很低。BytesRef就可以极大缓解这个问题。但有时候我们读取可能会跨block读取，涉及两个数组，还是需要将目标数据拷贝生成新数组，然后被BytesRef引用。BytesRef内部还提供了对其表示的字节数组进行转码的操作

# 2.属性

```java
/** 字节数组引用,大多情况下是一个buffer的引用，当需要表示跨buffer数据时，则引用一个手动生成的数组 */
public byte[] bytes;
/** 在bytes中的偏移量 */
public int offset;
/** 占用的长度 */
public int length;
```

# 3.主要方法

## 3.1转成字符串

从bytes中按照offset和length进行截取，转成字符串。

需要注意的是BytesRef内部字符串都是进行UTF-8编码成字节数组进行存储的，但是JVM内部字符串是UTF16,故还需将UTF-8转成UTF16

```java
public String utf8ToString() {
    final char[] ref = new char[length];
  	//JVM内部字符串是utf16
    final int len = UnicodeUtil.UTF8toUTF16(bytes, offset, length, ref);
    return new String(ref, 0, len);
}
```

## 3.2浅拷贝

```java
@Override
public BytesRef clone() {
    return new BytesRef(bytes, offset, length);
}
```



## 3.3深拷贝

byte数组复制，不是复制引用

```java
public static BytesRef deepCopyOf(BytesRef other) {
    return new BytesRef(ArrayUtil.copyOfSubArray(other.bytes, other.offset, other.offset + other.length), 0, other.length);
}
```

