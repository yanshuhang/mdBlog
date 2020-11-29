# 缺失的最小正数

## 问题描述

给你一个未排序的整数数组，请你找出其中没有出现的最小的正整数。

## 解答

对于一个长度为n的数组来说,缺失的最小正数必在1到n+1之间  
遍历数组将数字移动到其值-1的下标位置处,例如:1移动到nums[0],2移动到nums[1]处  
再次遍历数组如果下标处的值不等于下标+1那么就找到了这个最小正数

``` java
public int solution(int[] nums) {
    int len = nums.length;
    for (int i = 0; i < len; i++) {
        while (nums[i] > 0 && nums[i] <= len && nums[i] != nums[nums[i] - 1]) {
            int temp = nums[nums[i] - 1];
            nums[nums[i] - 1] = nums[i];
            nums[i] = temp;
        }
    }
    for (int i = 0; i < len; i++) {
        if (nums[i] != i + 1) {
            return i+1;
        }
    }
    return len + 1;
}
```
