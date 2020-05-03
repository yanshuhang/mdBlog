# 最接近的三数之和

## 问题描述

给定一个包括 n 个整数的数组 nums 和 一个目标值 target。找出 nums 中的三个整数，使得它们的和与 target 最接近。返回这三个数的和。假定每组输入只存在唯一答案。

``` text
例如，给定数组 nums = [-1，2，1，-4], 和 target = 1.

与 target 最接近的三个数的和为 2. (-1 + 2 + 1 = 2).
```

## 解答

跟`三数之和为0`的核心思路一致：

1. 排序数组
2. 取一个值用于保留当前最接近的三数之和，作为遍历中比较的第一个值，可以取数组中３个数的和或者使用`Integer.MIN_VALUE`
3. 遍历数组，对数组中每个值进行求解：
    * 在该值的右侧取最小索引left的值和最大索引right的值求三数之和
    * 跟当前最接近的数比较，如果最接近则替换
    * 比较之后判断:如果和大于target,right索引左移,否则right索引右移

可优化的部分:

1. 在循环之前比较target跟数组中的三数之和的最小值、最大值进行比较，如果小于最小值或大于最大值则返回最小值、最大值
2. 在遍历中如果当前的三数之和等于target，直接返回
3. 在遍历中也进行`优化1`中的比较，如果出现`优化1`中的情况，则可以跳出本次遍历

``` java
public static int solution(int[] nums, int target) {
    // 数组排序
    Arrays.sort(nums);
    int len = nums.length;
    // 起始的值
    int result = nums[0] + nums[1] + nums[2];

    // target跟数组的三数之和的最小值和最大值比较
    // target比最小值还小，那最小值就是最接近的
    if (target <= result) {
        return result;
    }
    // target比最大值还大，那最大值就是最接近的
    if (target >= nums[len - 3] + nums[len - 2] + nums[len - 1]) {
        return nums[len - 3] + nums[len - 2] + nums[len - 1];
    }

    // 对于每个值，遍历求解
    for (int i = 0; i < len - 2; i++) {
        // 如果当前值已经在之前求解过了，可以跳过本次求解
        if (i > 0 && nums[i] == nums[i - 1]) {
            continue;
        }

        int left = i + 1;
        int right = len -1;

        while (left != right) {
            int iterval = Math.abs(result - target);

            // 当前的三数之和的最小值，比较逻辑跟前面的一样
            int min = nums[i] + nums[left] + nums[left + 1];
            if (target < min) {
                if (Math.abs(min - target) < iterval) {
                    result = min;
                }
                break;
            }
            int max = nums[i] + nums[right] + nums[right - 1];
            if (target > max) {
                if (Math.abs(max - target) < iterval) {
                    result = max;
                }
                break;
            }

            int sum = nums[i] + nums[left] + nums[right];
            // 如果相等，直接返回
            if (target == sum) {
                return sum;
            }

            // 比较如果更接近，更换reslut值
            if (Math.abs(sum - target) < iterval) {
                result = sum;
            }

            // 如果和比target大，right右移，在下次while循环中sum会变小，判断其是否更接近
            // else中是一样的逻辑
            if (sum > target) {
                right--;
            } else {
                left++;
            }
        }
    }
    return result;
}
```
