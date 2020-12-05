# 跳跃游戏

## 问题描述

给定一个非负整数数组，你最初位于数组的第一个位置。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

判断你是否能够到达最后一个位置。

## 解答

遍历数组,维护一个变量保存可以跳到的最大位置,在最大位置内的都可以跳到,如果到最后最大位置大于等于数组长度-1,那么就可以跳到最后一个位置

``` java
public boolean solution1(int[] nums) {
    int max = 0;
    for (int i = 0; i < nums.length; i++) {
        // 如果当前索引不在最大位置以内,后面直接就跳不到了,所以返回false
        if (i > max) {
            return false;
        }
        // 更新可以跳到的最大位置
        max = Math.max(max, i + nums[i]);
        // 最大位置大于等于数组长度-1,可以提前结束循环
        if (max >= nums.length - 1) {
            return true;
        }
    }
    return true;
}
```
