# 在排序数组中查找元素的第一个和最后一个位置

## 问题描述

给定一个按照升序排列的整数数组 nums，和一个目标值 target。找出给定目标值在数组中的开始位置和结束位置。

你的算法时间复杂度必须是 O(log n) 级别。

如果数组中不存在目标值，返回 [-1, -1]。

## 解答

排序数组中查找使用二分法,但不会在第一次匹配时停止,每次记录下本次匹配的索引

* 查找第一个时,在每次匹配后,继续在前半段匹配
* 查找最后一个时,在每次匹配后,继续在后半段匹配
* 在循环结束后返回最后记录下的索引

``` java
public static int[] solution(int[] nums, int target) {
    int left = findBound(nums, target, true);
    int right = findBound(nums, target, false);
    return new int[]{left, right};
}
private static int findBound(int[] nums, int target, boolean b) {
    int left = 0;
    int right = nums.length - 1;
    int res = -1;
    while (left <= right) {
        int mid = (left + right) / 2;
        if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] < target) {
            left = mid + 1;
        // 在匹配时,通过boolean判断是往前还是往后继续匹配
        } else if (b) {
            res = mid;
            right = mid - 1;
        } else {
            res = mid;
            left = mid + 1;
        }

    }
    return res;
}
```
