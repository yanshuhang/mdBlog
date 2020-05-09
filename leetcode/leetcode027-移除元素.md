# 移除元素

## 问题描述

给你一个数组 nums 和一个值 val，你需要 原地 移除所有数值等于 val 的元素，并返回移除后数组的新长度。

不要使用额外的数组空间，你必须仅使用 O(1) 额外空间并 原地 修改输入数组。

元素的顺序可以改变。你不需要考虑数组中超出新长度后面的元素。

示例 1:

``` java
给定 nums = [3,2,2,3], val = 3,

函数应该返回新的长度 2, 并且 nums 中的前两个元素均为 2。

你不需要考虑数组中超出新长度后面的元素。
```

## 解答

双指针：快指针指向下一个判断的值，慢指针指向已经判断过符合的值的下一个位置

* 不等时，快慢指针都加1
* 相等时，快指针+1,慢指针不动

``` java
public static int solution(int[] nums, int val) {
    int result = 0;
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] != val) {
            nums[result++] = nums[i];
        }
    }
    return result;
}
```
