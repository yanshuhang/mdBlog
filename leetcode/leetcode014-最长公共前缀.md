# 最长公共前缀

## 问题描述

编写一个函数来查找字符串数组中的最长公共前缀。

如果不存在公共前缀，返回空字符串 ""。

示例 1:

``` text
输入: ["flower","flow","flight"]
输出: "fl"
```

示例 2:

``` text
输入: ["dog","racecar","car"]
输出: ""
解释: 输入不存在公共前缀。
```

说明:

所有输入只包含小写字母 a-z 。

## 解答

暴力匹配法，感觉挺sb的，提交显示执行只用了1ms……

``` java
public static String solution(String[] strs) {
    if (strs == null || strs.length == 0) {
            return "";
    }
    StringBuilder stringBuilder = new StringBuilder();
    // 遍历数组第一个字符串的字符
    for (int i = 0; i < strs[0].length(); i++) {
        // 遍历数组中其余的字符串
        for (int j = 1; j < strs.length; j++) {
            // 如果字符串长度过小，或者字符不匹配，返回当前的stringBuilder
            // 长度的判断放在前面，避免后面的索引越界
            if (strs[j].length() < i + 1 || strs[j].charAt(i) != strs[0].charAt(i)) {
                return stringBuilder.toString();
            }
        }
        // 所有字符串都匹配了该字符，拼接到stringBuilder中
        stringBuilder.append(strs[0].charAt(i));
    }
    return stringBuilder.toString();
}
```

可以不使用stringbuilder，记录下i的值，使用substring返回即可

``` java
public static String solution1(String[] strs) {
    if (strs == null || strs.length == 0) {
        return "";
    }
    for (int i = 0; i < strs[0].length(); i++) {
        char c = strs[0].charAt(i);
        for (int j = 1; j < strs.length; j++) {
            if (strs[j].length() == i || strs[j].charAt(i) != c) {
                return strs[0].substring(0, i);
            }
        }
    }
    return strs[0];
}
```

一个0ms的解法：遍历数组，判断第一个字符串是否是前缀，如果不是：将第一个字符串的最后一个字符去掉，直到是本次遍历的字符串的前缀为止，然后继续往下遍历数组。

``` java
 public String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++)
        while (strs[i].indexOf(prefix) != 0) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    return prefix;
}
```
