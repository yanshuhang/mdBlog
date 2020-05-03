# 正则表达式匹配

## 问题描述

给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

``` text
'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
```

所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。

说明:

``` text
s 可能为空，且只包含从 a-z 的小写字母。
p 可能为空，且只包含从 a-z 的小写字母，以及字符 . 和 *。
```

## 解答

### 暴力匹配，使用递归进行回溯

如果只需两个字符串比较，遍历比较即可：

``` java
if(s.charAt(i) == p.charAt(i))
```

如果正则表达式只有 '.' 一个符号，也只需遍历比较即可：

``` java
if(s.charAt(i) == p.charAt(i) || p.charAt(i) == '.')
```

由于 '*' 符号的特殊匹配：匹配0个或多个之前的一个元素

1. 匹配0个：保持s不变，p减去前两个字符，继续递归
2. 匹配多个：首先判断首字符是否匹配，如果匹配，s减去首字符，继续递归

综合以上两种情况进行判断

``` java
public static boolean isMatch(String s, String p) {
    if (p.isEmpty()) {
        return s.isEmpty();
    }
    // 判断首字符是否匹配
    boolean firstMatch = (!s.isEmpty() && (p.charAt(0) == s.charAt(0) || p.charAt(0) == '.'));
    // 遇到 '*' 符号的两种情况的递归结果取或
    if (p.length() >= 2 && p.charAt(1) == '*') {
        return (isMatch(s, p.substring(2)) || (firstMatch && isMatch(s.substring(1), p)));
    } else {
        // 非 '*' 的情况，在首字符匹配时继续递归
        return firstMatch && isMatch(s.substring(1), p.substring(1));
    }
}
```

### 动态规划

创建dp[s.length+1][p.length+1]的数组，存储暴力法中递归子过程的结果，避免暴力法中递归的重复计算  
dp[0][0]表示匹配完的结果，dp[s.length][p.length]表示s和p为空时的匹配结果  
核心的判断逻辑和暴力法相同

``` java
public static boolean isMatch1(String s, String p) {
    boolean[][] dp = new boolean[s.length() + 1][p.length() + 1];
    // s、p都为空返回true
    dp[s.length()][p.length()] = true;
    // 自底向上模拟递归的回溯，将结果存储在dp数组中
    for (int i = s.length(); i >= 0; i--) {
        for (int j = p.length() - 1; j >= 0; j--) {
            boolean firstMatch = (i > 0 && (s.charAt(i) == p.charAt(j) || p.charAt(j) == '.'));
            if (j + 1 < p.length() && p.charAt(j + 1) == '*') {
                // 当i=s.length()时，firstMatch会使dp[i+1][j]短路，所以不会出现数组越界的问题
                dp[i][j] = dp[i][j + 2] || firstMatch && dp[i + 1][j];
            } else {
                dp[i][j] = firstMatch && dp[i+1][j+1];
            }
        }
    }

    return dp[s.length()][p.length()];
}
```
