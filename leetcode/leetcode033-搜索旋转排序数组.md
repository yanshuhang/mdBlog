# 搜索旋转排序数组

## 问题描述

给你一个整数数组 nums ，和一个整数 target 。

该整数数组原本是按升序排列，但输入时在预先未知的某个点上进行了旋转。（例如，数组 [0,1,2,4,5,6,7] 可能变为 [4,5,6,7,0,1,2] ）。

请你在数组中搜索 target ，如果数组中存在这个目标值，则返回它的索引，否则返回 -1 。

## 解答

二分查找,由于数组不是正常的排序数组,比较之后target可能出现在相反的一半,需要多一层判断

``` java
public static int solution(int[] nums, int target) {
    if (nums == null || nums.length == 0) {
        return -1;
    }
    if (nums.length == 1) {
        if (nums[0] == target) {
            return 0;
        } else {
            return -1;
        }
    }
    int left = 0;
    int right = nums.length - 1;
    while (left <= right) {
        int mid = (left + right) / 2;
        if (nums[mid] == target) {
            return mid;
        }
        if (nums[0] <= nums[mid]) {
            if (nums[0] <= target && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {
            if (nums[mid] < target && target <= nums[nums.length - 1]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    return -1;
}
```
