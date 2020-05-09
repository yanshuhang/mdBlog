# 多数元素

## 问题描述

给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数大于 ⌊ n/2 ⌋ 的元素。

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

示例 1:

``` text
输入: [3,2,3]
输出: 3
```

示例 2:

``` text
输入: [2,2,1,1,1,2,2]
输出: 2
```

## 解答

排序：由于多数元素数量大于一半，那么排序之后索引n/2处的元素必为多数元素

``` java
public static int solution1(int[] nums) {
        Arrays.sort(nums);
        return nums[nums.length / 2];
    }
```

使用hashMap：记录每个元素出现的次数，遍历map发现次数大于n/2的返回即可

``` java
Map<Integer, Integer> map = new HashMap<>();
        for (int num : nums) {
            map.put(num, map.getOrDefault(num, 0) + 1);
        }
        int half = nums.length/2;
        int ans = 0;
        for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
            if (entry.getValue() > half) {
                ans = entry.getKey();
                break;
            }
        }
        return ans;
```
