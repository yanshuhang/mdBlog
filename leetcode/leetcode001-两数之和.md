# 两数之和

## 问题描述

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

``` text
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9  
所以返回 [0, 1]
```

## 解答

遍历数组，使用map存储`当前值的补数和下标`对应的key-value对，如果map中存在当前值的key，则找到了和为target的两个数，返回该key对应的value和当前值的下标。

``` java
public static int[] twoSum(int[] nums, int target) {
    HashMap<Integer, Integer> map = new HashMap<>();
    int[] result = new int[2];
    for (int i = 0; i < nums.length; i++) {
        if (!map.containsKey(nums[i])) {
            map.put(target - nums[i], i);
        } else {
            result[0] = map.get(nums[i]);
            result[1] =i;
        }
    }
    return result;
}
```
