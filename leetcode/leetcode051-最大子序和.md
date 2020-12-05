# 最大子序和

## 问题描述

给定一个整数数组 nums ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

## 解答

遍历数组,将数字相加:

1. 如果sum小于等0,它加上面的值不会让结果变大,所以省略sum,将其置为当前值
2. sum大于0,则加上当前值
3. 在最终结果和当前sum中取最大值

``` java
public int solution(int[] nums) {
    int ans = nums[0];
    int sum = 0;
    for (int num : nums) {
        if (sum > 0) {
            sum += num;
        } else {
            sum = num;
        }
        ans = Math.max(sum, ans);
    }
    return ans;
}
```
