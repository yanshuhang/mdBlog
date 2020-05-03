# 三数之和

## 问题描述

给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有满足条件且不重复的三元组。

注意：答案中不可以包含重复的三元组。

示例：

``` text
给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

## 解答

1. 排序数组
2. 从左侧开始遍历，对于每个值在其右侧进行求解
3. 求解过程：选取右侧的最小值和最大值，如果和偏大最大值往左找，偏小则最小值向右找
4. 遍历其右侧求解，找到就加入到result，然后继续在右侧求解
5. 可以加速的逻辑：
    * 当遍历第一个值大于0时即可返回result，因为其右侧的都大于0 不可能有合适的
    * 如果遍历的第一个值和上一次的值相同，跳过
    * 当找到一个组合之后，判断最小值和最大值的相邻的值是否相等，相等则可以跳过，这样可以去重

``` java
public List<List<Integer>> solution(int[] nums) {
    if (nums == null || nums.length <= 2) {
        return Collections.emptyList();
    }
    List<List<Integer>> result = new LinkedList<>();
    Arrays.sort(nums);
    for (int i = 0; i < nums.length - 2; i++) {
        if (nums[i] > 0) {
            return result;
        }
        if (i != 0 && nums[i] == nums[i - 1]) {
            continue;
        }
        int left = i + 1;
        int right = nums.length - 1;
        while (left < right) {
            int sum = -(nums[left] + nums[right]);
            if (sum == nums[i]) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                while (left < right && nums[left] == nums[left+1]) {
                    left++;
                };
                while (left < right && nums[right] == nums[right - 1]) {
                    right--;
                }
            }
            if (sum < nums[i]) {
                right--;
            } else {
                left++;
            }
        }
    }
    return result;
}
```
