# 最长无重复字符子串

## 问题描述

给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。

示例 1:

``` text
输入: "abcabcbb"
输出: 3
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

示例 2:

``` text
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

示例 3:

``` text
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

## 解答

使用map记录字符及其最新的下标(因为有重复)，使用i、j遍历字符串，i表示当前无重复字符子串的开始位置，j是当前位置，如果j处字符在map中存在且该字符对应的value大于i，则说明[i,j]子串有重复了，将i的位置移到j+1，继续遍历

result记录最长的无重复子串的长度，每次遍历都取上次的result值和当前长度的最大值

``` java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        // 记录字符及其下标
        Map<Character, Integer> map = new HashMap<>();
        int n = s.length();
        int result = 0;
        // 遍历字符串，将其记录在map中，每个字符只存储一个key
        // i是不重复连续子串的开始位置，j是当前位置
        for (int i = 0, j = 0; j < n; j++) {
            // 如果当前字符在map中存在，且其下标在i与j之间
            // 将i移到map中存储的下标+1
            if (map.containsKey(s.charAt(j)) && map.get(s.charAt(j)) >= i) {
                i = map.get(s.charAt(j)) + 1;
            }
            // result记录了之前的最长子串、与当前的比较取最大的
            result = Math.max(result, j - i + 1);
            // 更新该字符出现的下标
            map.put(s.charAt(j), j);
        }
        return result;
    }
}
```
