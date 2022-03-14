---
title: Apache工具类源码阅读
date: 2019-05-08 21:37:08
tags:
---
## Intro
常用的有commons-lang3 commons-io commons-collections，工具类并没有复杂的结构，阅读起来简单。很适合学习优秀的代码习惯和细节。文中简要挑一些类说明。

## commons-lang3
这个包主要扩展了java.lang，提供一些常用操作。

#### StringUtils
`String`是不可变的，因此对String的修改操作都是返回新建的字符串。此外，类中用到了需要变长参数，如`isAnyEmpty(final CharSequence... css)`，可以避免调用时需要创建数组或者List。本质上是一个语法糖，由调用时创建数组传入。
##### 去除
* trim 
* truncate 
* strip

传入目标str和stripStr，将str头尾中在stripStr中出现的字符去除，stripStr为null默认去除空格。
```java
public static String stripStart(final String str, final String stripChars) {
    int strLen; // str长度
    if (str == null || (strLen = str.length()) == 0) { // 判断时赋值
        return str;
    }
    int start = 0; // 第一个不在stripChars中的字符
    if (stripChars == null) { // 去除空格的情况
        while (start != strLen && Character.isWhitespace(str.charAt(start))) {
            start++;
        }
    } else if (stripChars.isEmpty()) { // 不去除。没有这个判断也可以执行，用于避免搜一遍字符串。
        return str;
    } else {
        while (start != strLen && stripChars.indexOf(str.charAt(start)) != INDEX_NOT_FOUND) {
            start++;
        }
    }
    return str.substring(start);
}
```
##### 比较
* equals
* compare
  
都是传入两个或以上的字符串比较，考虑了空和null的情况。之后调用原生api。
##### 查找
* indexOf
* ordinalIndexOf
* contains
* indexOfAnyBut
* containsAny

```java
// 查找出现第ordinal次searchStr的位置
private static int ordinalIndexOf(final CharSequence str, final CharSequence searchStr, final int ordinal, final boolean lastIndex) {
    if (str == null || searchStr == null || ordinal <= 0) {
        return INDEX_NOT_FOUND; // -1
    }
    if (searchStr.length() == 0) { // 查找的串为空
        return lastIndex ? str.length() : 0;
    }
    int found = 0;
    // set the initial index beyond the end of the string
    // this is to allow for the initial index decrement/increment
    int index = lastIndex ? str.length() : INDEX_NOT_FOUND;
    do {
        if (lastIndex) {
            index = CharSequenceUtils.lastIndexOf(str, searchStr, index - 1); // step backwards thru string
        } else {
            index = CharSequenceUtils.indexOf(str, searchStr, index + 1); // step forwards through string
        }
        if (index < 0) {
            return index; // 没找到or数量不够
        }
        found++;
    } while (found < ordinal);
    return index;
}
```

##### 子串/修改
* substring
* left / mid / right
* substringBefore / substringAfter / substringBetween
* split
* join
* removeStart
* replace
* replaceChars
* repeat
* rotate
* reverse
* abbreviate
* wrap

```java
private static String[] splitByWholeSeparatorWorker(
        final String str, final String separator, final int max, final boolean preserveAllTokens) {
    //... null空判断省略

    final int separatorLength = separator.length();

    final ArrayList<String> substrings = new ArrayList<>();
    int numberOfSubstrings = 0;
    int beg = 0; // 开始位置标记
    int end = 0; // 结束位置标记
    while (end < len) {
        end = str.indexOf(separator, beg);

        if (end > -1) { // 找到
            if (end > beg) { // ！begin是否等于end，也即处理两个speratorStr相连
                numberOfSubstrings += 1;

                if (numberOfSubstrings == max) { // 数量达到max，子串取到尾
                    end = len;
                    substrings.add(str.substring(beg));
                } else {
                    substrings.add(str.substring(beg, end));
                    beg = end + separatorLength; // 下次搜素起始位置
                }
            } else {
                // We found a consecutive occurrence of the separator, so skip it.
                if (preserveAllTokens) { // 保留两个相连speartorStr之间的空串EMPTY("")
                    numberOfSubstrings += 1;
                    if (numberOfSubstrings == max) {
                        end = len;
                        substrings.add(str.substring(beg));
                    } else {
                        substrings.add(EMPTY);
                    }
                }
                beg = end + separatorLength;
            }
        } else {
            // String.substring( beg ) goes from 'beg' to the end of the String.
            substrings.add(str.substring(beg));
            end = len;
        }
    }
    return substrings.toArray(new String[substrings.size()]); // list转数组方式
}
```
p.s 我感觉splitByWholeSeparatorWorker和splitWorker的功能一样啊……


#### ArrayUtils
* toString
```java
public static String toString(final Object array, final String stringIfNull) {
    if (array == null) {
        return stringIfNull;
    }
    return new ToStringBuilder(array, ToStringStyle.SIMPLE_STYLE).append(array).toString();
}
```
方法很简单，但是逻辑很有意思。每次都会创建一个`ToStringBuilder`，其数组元素的组合依赖`ToStringStyle`，包括字符串开头结尾和连接等，具体有很多。ToStringStyle也提供了一些常用组合的现成静态类。`append`方法最终会调用`ToStringStyle`的`ToStringStyle`方法，这里有个trick，为了防止数组循环调用appendInternal，`ToStringStyle`中维持了一个`ThreadLocal<WeakHashMap<Object, Object>>`。每次执行
`ToStringStyle`前需要先检查是否注册了array对象，未注册需先注册，即往WeakHashMap中put。put的参数为arry和null。这个ThreadLocal被private static修饰，保证了ThreadLocal内存不会泄漏并使map存放多个线程的array。WeakHashMap则可以使退出调用的array可以被垃圾回收，因为value为null，所以保证了WeakHashMap也不会发生内存泄漏。事实上在方法调用完成后会主动调用unregister把map中的array给remove掉，因此我觉得可能这里weakhaspmap意义不大。