# 字母异位词分组

## 问题描述

给定一个字符串数组，将字母异位词组合在一起。字母异位词指字母相同，但排列不同的字符串。

## 解答

字母异位词再排序后是一样的,那么将字符串排序后作为hashmap的key,vaule对应一个列表存储原始的字符串

``` java
public List<List<String>> solution(String[] strs) {
    if (strs == null || strs.length == 0) {
        return Collections.emptyList();
    }
    Map<String, List<String>> map = new HashMap<>();
    for (String str : strs) {
        char[] chars = str.toCharArray();
        Arrays.sort(chars);
        String key = String.valueOf(chars);
        if (!map.containsKey(key)) {
            map.put(key, new ArrayList<>());
        }
        map.get(key).add(str);
    }
    return new ArrayList<>(map.values());
}
```
