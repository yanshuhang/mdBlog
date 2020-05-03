# 电话号码的字母组合

## 问题描述

给定一个仅包含数字 2-9 的字符串，返回所有它能表示的字母组合。

给出数字到字母的映射如下（与电话按键相同）。注意 1 不对应任何字母。

``` text
示例:

输入："23"
输出：["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"].
```

## 解答

使用递归回溯，相当于树的深度遍历，记录下树的所有路径

``` java
public class LetterCombinations {

    // 对应关系：数组下标对应数字，内容对于字符
    String[] digitsToString = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
    List<String> ans = new ArrayList<>();

    public List<String> solution(String digits) {
        if (digits.length() == 0) {
            return Collections.emptyList();
        }
        StringBuilder sb = new StringBuilder();
        char[] digitstr = digits.toCharArray();
        combine(sb, digitstr, 0);
        return ans;
    }

    /**
     *
     * @param sb： 用来拼接字符
     * @param digits： 数字字符数组
     * @param k： 当前在拼接第几个数字对于的字符
     */
    public void combine(StringBuilder sb, char[] digits, int k) {
        // 已拼接完成
        if (k == digits.length) {
            ans.add(sb.toString());
            return;
        }

        // 取得当前数字对于的字符数组
        int digit = digits[k] - '0';
        char[] chars = digitsToString[digit].toCharArray();

        // 遍历数组
        // 1.拼接字符
        // 2.递归
        // 3.递归返回删除k处的字符，该字符就是刚刚拼接的字符，进行下次for循环
        for (char c : chars) {
            sb.append(c);
            combine(sb, digits, k+1);
            sb.deleteCharAt(k);
        }
    }
}
```
