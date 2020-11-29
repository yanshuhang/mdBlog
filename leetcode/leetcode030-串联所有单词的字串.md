# 串联所有单词的子串

## 问题描述

给定一个字符串 s 和一些长度相同的单词 words。找出 s 中恰好可以由 words 中所有单词串联形成的子串的起始位置。

注意子串要与 words 中的单词完全匹配，中间不能有其他字符，但不需要考虑 words 中单词串联的顺序。

## 解答

暴力法:遍历两遍,比较字符串,由于不要求顺序,把字符串中所有单词出现的次数存到hashmap中,比较两个map是否相等

``` java
public static List<Integer> solution(String s, String[] words) {
    if (s == null || s.length() == 0 || words == null || words.length == 0) {
        return Collections.emptyList();
    }

    // 存储所有的符合条件的索引
    List<Integer> ans = new ArrayList<>();
    int word_len = words[0].length();
    int all_len = word_len * words.length;

    // 把数组中单词的出现次数存储到map中
    HashMap<String, Integer> map = new HashMap<>();
    for (String word : words) {
        map.put(word, map.getOrDefault(word, 0) + 1);
    }

    // 遍历所有情况,比较两个map是否相等
    for (int i = 0; i < s.length() - all_len + 1; i++) {
        HashMap<String, Integer> temp = new HashMap<>();
        for (int k = i; k < i + all_len; k = k + word_len) {
            String w = s.substring(k, k + word_len);
            temp.put(w, temp.getOrDefault(w, 0) + 1);
            // 如果相应的单词已经超过了数组中出现的次数,后面不用遍历了直接退出
            if (temp.getOrDefault(w, 0) > map.getOrDefault(w, 0)) {
                break;
            }
        }
        if (map.equals(temp)) {
            ans.add(i);
        }

    }
    return ans;
}
```

暴力法中不同次的遍历中,同一个单词可能重复的放入了map中,可优化的点:

* 每次遍历一个单词的长度
* 从后向前遍历,如果后面的失败,前面的也不用判断了,直接跳过
* 不同比较两个map是否相等,如果判断word_num个单词都没有失败,说明就成功了

``` java
public List<Integer> solution1(String s, String[] words) {
    if (s == null || s.length() == 0 || words == null || words.length == 0) {
        return Collections.emptyList();
    }

    List<Integer> ans = new ArrayList<>();
    int word_len = words[0].length();
    int word_num = words.length;
    int all_len = word_len * word_num;

    Map<String, Integer> map = new HashMap<>();
    for (String word : words) {
        map.put(word, map.getOrDefault(word, 0) + 1);
    }

    // for循环里的遍历,每次遍历一个单词的长度
    for (int i = 0; i < word_len; i++) {
        int start = i;
        while (start + all_len <= s.length()) {
            Map<String, Integer> temp = new HashMap<>();
            String subStr = s.substring(start, start + all_len);
            int k = word_num;
            while (k > 0) {
                // 从后向前比较
                String word = subStr.substring((k - 1) * word_len, k * word_len);
                int count = temp.getOrDefault(word, 0) + 1;
                if (count > map.getOrDefault(word, 0)) {
                    break;
                }
                temp.put(word, count);
                --k;
            }
            if (k == 0) {
                ans.add(start);
            }
            // 索引跳到失败的地方
            start = start + word_len * Math.max(k, 1);
        }
    }
    return ans;
}
```
