# 最长有效括号

## 问题描述

给定一个只包含 '(' 和 ')' 的字符串，找出最长的包含有效括号的子串的长度。

## 解答

判断括号的有效性,一般使用栈,提前放入一个下标:-1

* 遇到`(`,将其下标放入栈中
* 遇到`)`,弹出栈顶

  * 如果栈不为空,说明括号匹配了,判断是否是最长的
  * 如果栈为空,说明弹出了栈底的占位的下标,就是没有匹配成功，将下标压入

``` java
public static int solution(String s) {
    int maxlen = 0;
    LinkedList<Integer> stack = new LinkedList<>();
    stack.push(-1);
    char[] str = s.toCharArray();
    for (int i = 0; i < str.length; i++) {
        if (str[i] == '(') {
            stack.push(i);
        } else {
            stack.pop();
            if (stack.isEmpty()) {
                stack.push(i);
            } else {
                // 每次匹配的判断是否是最长的
                maxlen = Math.max(maxlen, i - stack.peek());
            }
        }
    }
    return maxlen;
}
```
