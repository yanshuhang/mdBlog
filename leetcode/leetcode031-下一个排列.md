# 下一个排列

## 问题描述

实现获取 下一个排列 的函数，算法需要将给定数字序列重新排列成字典序中下一个更大的排列。

如果不存在下一个更大的排列，则将数字重新排列成最小的排列（即升序排列）。

必须 原地 修改，只允许使用额外常数空间。

## 解答

1. 将左边的`较小数`与右边的`较大数`交换,此时数字有变大
2. 将`较大数`后面的数字按升序排列,此时得到的就是最终结果

``` java
public static void solution(int[] nums) {
    int i = nums.length - 2;
    // 如果数字比左边的小,交换才会变大,找到要交换的位置较小数
    while (i >= 0 && nums[i] >= nums[i + 1]) {
        i--;
    }
    // 在较小数左边找到比较小数大的最小数,交换
    if (i >= 0) {
        int k = nums.length - 1;
        while (k >= 0 && nums[k] <= nums[i]) {
            k--;
        }
        swap(nums, i, k);
    }
    // 然后反转后面的数
    reverse(nums, i + 1);
}

public static void swap(int[] nums, int a, int b) {
    int temp = nums[a];
    nums[a] = nums[b];
    nums[b] = temp;
}

public static void reverse(int[] nums, int start) {
    int left = start;
    int right = nums.length;
    while (left < right) {
        swap(nums, left, right);
        left++;
        right--;
    }
}
```
