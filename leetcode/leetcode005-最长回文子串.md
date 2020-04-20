# 最长回文子串

## 问题描述

给定一个字符串 s，找到 s 中最长的回文子串。你可以假设 s 的最大长度为 1000。

回文就是正读和反读一致的字符串

示例 1：

``` text
输入: "babad"
输出: "bab"
注意: "aba" 也是一个有效答案。
```

示例 2：

``` text
输入: "cbbd"
输出: "bb"
```

## 解答

中心扩展法：遍历字符串，每个位置从中心向两边扩展判断是否是回文

* 中心扩展时，先找到中间部分(最中心相同字符的部分)，然后再向两边扩展判断

``` java
public static String solution(String s) {
    if (s == null || s.length() == 0) {
        return "";
    }
    // 使用int数组保存当前最长回文的最低位和最高位
    int[] range = {0, 0};
    char[] chars = s.toCharArray();
    // 遍历字符串每个位置，中心扩展
    for (int i = 0; i < s.length(); i++) {
        i = centerExpand(chars, i, range);
    }
    return s.substring(range[0], range[1] + 1);
}

// 中心扩展
public static int centerExpand(char[] str, int low, int[] range) {
    int high = low;
    // 回文的中间部分是相同的字符，可以回避奇偶的判断
    while (high < str.length - 1 && str[high + 1] == str[low]) {
        high++;
    }
    // 记录下回文的中间部分的高位，这样外围遍历可以跳过一部分
    int ans = high;
    // 向中心位置两边扩展
    while (low > 0 && high < str.length - 1 && str[low - 1] == str[high + 1]) {
        low--;
        high++;
    }
    // 与记录中的回文比较长度，取最长的记录
    if (high - low > range[1] - range[0]) {
        range[0] = low;
        range[1] = high;
    }
    return ans;
}
```
