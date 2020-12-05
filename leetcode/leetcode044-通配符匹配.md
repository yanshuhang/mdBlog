# 通配符匹配

## 问题描述

给定一个字符串 (s) 和一个字符模式 (p) ，实现一个支持 '?' 和 '*' 的通配符匹配。

``` text
'?' 可以匹配任何单个字符。
'*' 可以匹配任意字符串（包括空字符串）。
```

两个字符串完全匹配才算匹配成功。

## 解答

### 动态规划

* 首先是一个保存状态的数组dp[][],保存s的前i个字符和p的前j个字符的匹配情况  
* 对数组进行初始化,对边界情况的判断
* 每个问题都依赖于它的子问题的解
  * 如果p中的字符是'?'或者直接匹配了s中的字符,那么dp[i][j] = dp[i-1][j-1]
  * 如果是'*',匹配0个或多个字符,那么dp[i][j] = dp[i][j-1] || dp[i-1][j]
* 这样遍历到最后dp[m][n]就是最终的结果了

``` java
public boolean solution(String s, String p) {
    int m = s.length();
    int n = p.length();
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    for (int i = 1; i <= n; i++) {
        if (p.charAt(i - 1) == '*') {
            dp[0][i] = true;
        } else {
            break;
        }
    }
    for (int i = 1; i <= m; i++) {
        char cc = s.charAt(i - 1);
        for (int j = 1; j <= n; j++) {
            char cp = p.charAt(j - 1);
            if (cp == '*') {
                dp[i][j] = dp[i-1][j] || dp[i][j-1];
            } else if (cp == cc || cp == '?') {
                dp[i][j] = dp[i-1][j-1];
            }
        }
    }
    return dp[m][n];
}
```

### 四指针法

i,j指向单个匹配的字符,istar,jstar指向'*'匹配的位置,优先单个字符匹配,在匹配失败时转为之前的星号多个字符

* 首先匹配单个字符,匹配成功i,j后移一位
* 如果上面匹配不成功再匹配星号,匹配成功istar指向跟星号匹配的字符,jstar指向星号,j后移一位,此时星号匹配0个字符
* 如果上面两个都匹配不成功,再判断之前是否有过星号匹配,如果有那么星号将向后匹配一位:istar,i后移一位,j指向星号后的下一个字符
* 如果都单个字符失败,当前也不是星号,之前也没有出现过星号,那么匹配失败
* 最后匹配完成后判断j是到达了最后一位还是星号,如果是星号必须其后面的全是星号才成功

``` java
public boolean solution1(String s, String p) {
    if (p == null || p.isEmpty()) {
        return s == null || s.isEmpty();
    }
    int i=0, j=0, istar = -1, jstar = -1,slen = s.length(), plen = p.length();
    while (i < slen) {
        // 优先单个字符匹配
        if (j < plen && (s.charAt(i) == p.charAt(j) || p.charAt(j) == '?')) {
            i++;
            j++;
        // 单个字符不匹配,但j是星号,星号先匹配0个,但记录下位置
        } else if (j < plen && p.charAt(j) == '*') {
            istar = i;
            jstar = j;
            j++;
        // 单个字符不匹配,j也不是星号,退到上一个星号,星号先匹配一个字符
        } else if (istar > -1) {
            i = ++istar;
            j = jstar + 1;
        // 都不成功那就是匹配失败了
        } else {
            return false;
        }
    }
    // 如果j指向星号,j后面的必须全是星号才行
    while (j < plen && p.charAt(j) == '*') {
        j++;
    }
    return j == plen;
}
```
