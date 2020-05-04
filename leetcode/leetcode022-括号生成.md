# 括号生成

## 问题描述

数字 n 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 有效的 括号组合。

示例：

``` text
输入：n = 3
输出：[
       "((()))",
       "(()())",
       "(())()",
       "()(())",
       "()()()"
     ]
```

## 解答

1. 第一个肯定是左括号
2. 如果剩余有左括号，放左括号是没有任何问题的
3. 在已经放的左括号多余右括号的情况下，也就是剩余的右括号比左括号多，才可以放右括号

``` java
ArrayList<String> list = new ArrayList<>();
public List<String> solution(int n) {
    dfs(n, n, "");
    return list;
}

public void dfs(int left, int right, String curStr) {
    // 左右括号都放完，方法返回
    // 最后一个肯定是右括号，所以一定是递归2返回
    // 然后一直回溯到递归1返回，然后继续递归2
    if (left == 0 && right == 0) {
        list.add(curStr);
        return;
    }
    // 递归1：有左括号就放左括号
    if (left > 0) {
        dfs(left -1, right, curStr + "(");
    }
    // 递归2：剩余的右括号更多时，可以放右括号
    if (right > left) {
        dfs(left, right - 1, curStr + ")");
    }
}
```
