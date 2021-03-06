# 是否位回文数

## 问题描述

判断一个整数是否是回文数。回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

示例 1:

``` text
输入: 121
输出: true
```

示例 2:

``` text
输入: -121
输出: false
解释: 从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。
```

示例 3:

``` text
输入: 10
输出: false
解释: 从右向左读, 为 01 。因此它不是一个回文数。
```

## 解答

将数字的后半部分反转取出，跟剩下的部分比较

* 原数字长度偶数：比较两部分是否相等
* 原数组长度奇数：新数字/10跟旧数字是否相等

``` java
public static boolean isPalindrome(int x) {
    // 小于0 或者以0结尾的都不是
    if (x < 0 || (x != 0 && x % 10 == 0)) return false;

    int res = 0;
    // 只需要遍历一般的数字长度即可
    while (x > res) {
        res = res * 10 + x % 10;
        x = x / 10;
    }
    return res == x || res/10 == x;
}
```
