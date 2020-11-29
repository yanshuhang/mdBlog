# 外观数列

## 问题描述

## 解答

每一个计算都是在上一次的结果基础上进行的,这里使用递归来解决:

* n为1和2时直接返回结果
* 其余皆在上次的结果上遍历计算:双指针pre和current
  * 如果当前跟前面的一样,增加count
  * 如果不一样,将pre和count记录,然后变换pre指向
  * 当遍历到最后一个字符时,不论是上面哪种情况,都会少记录一部分,所以需要判断下

``` java
public static String solution(int n) {
    if (n == 1) {
        return "1";
    }
    if (n == 2) {
        return "11";
    }
    String before = solution(n - 1);
    StringBuilder stringBuilder = new StringBuilder();
    int count = 1;
    char pre = before.charAt(0);
    for (int i = 1; i < before.length(); i++) {
        char current = before.charAt(i);
        if (current == pre) {
            count++;
        } else {
            stringBuilder.append(count).append(pre);
            count = 1;
            pre = current;
        }
        if (i == before.length() - 1) {
            stringBuilder.append(count).append(current);
        }
    }
    return stringBuilder.toString();
}
```
