---
title: Android中android:maxLength属性解析及使用字节数来限制EditText文字长度（汉字算2个字符，英文算一个字符）
date: 2018-11-28 14:03:23
tags: android
category: android
---
# 代码分析

　　Android中EditText有android:maxLength属性用来限制文字长度，不过其不区分中文与英文，一律按一个字符来计算长度，现在如果要求汉字算2个长度，英文及符号算1个长度，那么就需要自己实现了，在实现之前我们分析下maxLength源码，看下它是怎么做到限制文字长度的

## LengthFilter详解

　　首先找到EditText的源码，发现只有不足两百行代码，且无maxLength实现，那么肯定在其父类TextView里面，在TextView里面搜索maxLength，发现只有三处引用，很快就能找到如下代码：

```java
if (maxlength >= 0) {
    setFilters(new InputFilter[] { new InputFilter.LengthFilter(maxlength) });
} else {
    setFilters(NO_FILTERS);
}
```

　　点进LengthFilter的代码，其注释如下

      This filter will constrain edits not to make the length of the text greater than the specified length.

　　显然正是实现长度限制的，里面起作用的只有一个filter方法：

```java
public CharSequence filter(CharSequence source, int start, int end, Spanned dest,
        int dstart, int dend) {
    int keep = mMax - (dest.length() - (dend - dstart));
    if (keep <= 0) {
        return "";
    } else if (keep >= end - start) {
        return null; // keep original
    } else {
        keep += start;
        if (Character.isHighSurrogate(source.charAt(keep - 1))) {
            --keep;
            if (keep == start) {
                return "";
            }
        }
        return source.subSequence(start, keep);
    }
}
```

　　这个方法的主要作用就是把dest的dstart到dend之间的内容用source的start到end之间的内容替换。举个例子，假设输入框maxLength属性为5，里面的内容是“这是个例子”，如果此时你选中“例”字，然后在输入法里面输入了“大栗”，当你点击输入法候选栏里的“大栗”两个字时，分析此时下传进filter方法的各项参数：

- source是你输入的字，为“大栗”
- start为0，end为2，从source中取0-2之间的内容，即“大栗”两个字，意味着要用这两个字替换掉原字符串的一部分。这里如果不明白start、end是如何取值的，可以想象下两个字将一整个空行分为3个空，给每个空标上数字，从左到右依次为0、1、2，那么0到2之间刚好是“大栗”两个字
- dest是原字符串，为“这是个例子”，
- dstart值为3，dend值为4，dest的dstart到dend之间就是“例”字，意味着要把“例”字给替换掉。

综合起来意思就是要把dest(“这是个例子”)的3到4之间的内容(“例”)用source(“大栗”)的0到2之间的内容(“大栗”)替换，替换的结果是“这是个大栗子”，已经超过maxLength定义的5个字符，那么filter就要进行截取，现在就来看下它是如何进行截取的。

第一行

```java
    int keep = mMax - (dest.length() - (dend - dstart));
```

以上面举的例子来分析，dest.length()为5，dend-dstart为1，两者相减之后为4，这个其实就是dest(“这是个例子”)去掉dstart(3)到dend(4)之间的内容后剩下的字符的长度，然后mMax为5，减去4，keep值为1，所以keep的意思就是dest去掉dstart到dend之间的内容后，还剩下多少个可用长度。

接下来根据keep值大小分情况处理：

- 小于等于0时返回空字符串，意味着不进行添加操作，只删除dest的dstart到dend之间的内容
- 大于等于end - start时，意味着剩余空间足够，不作任何操作，返回null
- 剩下的情况就在source里面从start开始截取keep长度的字符串返回。

以上就是android:maxLength的实现原理，

# 代码实现

## 计算字符串长度

现在要实现本文开头说的需求，汉字当2个字符，英文算一个字符，只需要在计算字符串长度时使用自己的方法就行了，因为英文字符及特殊符号的ascii码都是小于128的，可以用如下方法计算字符串的字节长度：

```java
/** 计算字符串的字节数 */
private int byteLength(CharSequence cs) {
    int index = 0;
    int count = 0;

    while (index < cs.length()) {
        char c = cs.charAt(index++);
        count = c < 128 ? count + 1 : count + 2;
    }
    return count;
}
```

## 自定义InputFilter实现

最后定义ByteLengthFilter类，实现InputFilter接口，代码如下：

```java
/** 按照字节数来限制EditText的输入，参照maxLength属性实现，具体见{@link InputFilter.LengthFilter} */
public class ByteLengthFilter implements InputFilter {
    private final int mMax;

    public ByteLengthFilter(int max) {
        mMax = max;
    }

    /** 计算字符串的字节数 */
    private int byteLength(CharSequence cs) {
        int index = 0;
        int count = 0;

        while (index < cs.length()) {
            char c = cs.charAt(index++);
            count = c < 128 ? count + 1 : count + 2;
        }
        return count;
    }

    @Override
    public CharSequence filter(CharSequence source, int start, int end, Spanned dest, int dstart, int dend) {
//            Log.d("ByteLengthFilter", "src:" + source + " start:" + start + " end:" + end
//                    + " dest:" + dest + " dstart:" + dstart + " dend" + dend)

        //dest是改变前的字符，dstart到dend之间是dest中要被删除的字符，keep表示还剩多少可用字节数
        int keep = mMax - (byteLength(dest) - (byteLength(dest.subSequence(dstart, dend))));

        if (keep <= 0) {
            //可用字节数少于0时返回空字符串，代表dest的dstart-dend之间的字符串要被删除
            return "";
        } else if (keep >= byteLength(source.subSequence(start, end))) {
            //可用字节数充足时不做任何操作
            return null; // keep original
        } else {
            //source是将要替换到原字符串的，当keep小于source的字节数时，从source里面截取不大于keep字节长度的字符串
            int index = start;
            int count = 0;
            while (count < keep) {
                char c = source.charAt(index++);
                count = c < 128 ? count + 1 : count + 2;
            }

            if (count > keep) index--;

            return source.subSequence(start, index);
        }
    }
}
```

## 添加新InputFilter到EditText

最后，在Activity里面为了不破坏EditText已有的filter，可按如下方式添加新InputFilter：

```java
int max=10;
EditText et = findViewById(R.id.et);
//获取EditText的InputFilter数组
InputFilter[] filters = et.getFilters();
//创建新的InputFilter数组，长度是上面的filters长度加1
InputFilter[] newFilters = new InputFilter[filters.length + 1];
//把filters拷贝到newFilters里面
System.arraycopy(filters, 0, newFilters, 0, filters.length);
//将newFilters最后一个元素设置为上面定义的ByteLengthFilter
newFilters[filters.length] = new ByteLengthFilter(max);
//最后将newFilters设置回EditText
et.setFilters(newFilters);
```
