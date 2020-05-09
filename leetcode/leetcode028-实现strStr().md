# 实现strStr()

## 问题描述

实现 strStr() 函数。

给定一个 haystack 字符串和一个 needle 字符串，在 haystack 字符串中找出 needle 字符串出现的第一个位置 (从0开始)。如果不存在，则返回  -1。

示例 1:

``` text
输入: haystack = "hello", needle = "ll"
输出: 2
```

示例 2:

``` text
输入: haystack = "aaaaa", needle = "bba"
输出: -1
```

## 解答

遍历一遍比较是否有一样的子串：

* 首字符的匹配可以过滤很多错误答案
* 只需循环到两个字符串长度差的位置，后面的长度过短不可能匹配上

``` java
public static int solution(String haystack, String needle) {
    if (needle.length() == 0) {
        return 0;
    }
    int max = haystack.length() - needle.length();
    char first = needle.charAt(0);
    for (int i = 0; i <= max; i++) {
        // 寻找首字符匹配
        if (haystack.charAt(i) != first) {
            while (++i <= max && haystack.charAt(i) != first) {
            }
        }
        // 找到首字符匹配的位置，判断后面的字符是否匹配
        if (i <= max) {
            int j = i +1;
            int end = j + needle.length() -1;
            // 如果for循环结束，匹配的长度刚好则返回i
            // 否则继续外层for循环
            for (int k = 1; j < end && haystack.charAt(j) == needle.charAt(k); j++, k++) {

            }
            if (j == end) {
                return i;
            }
        }
    }
    return -1;
}
```
