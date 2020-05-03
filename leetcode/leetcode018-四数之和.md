# 四数之和

## 问题描述

给定一个包含 n 个整数的数组 nums 和一个目标值 target，判断 nums 中是否存在四个元素 a，b，c 和 d ，使得 a + b + c + d 的值与 target 相等？找出所有满足条件且不重复的四元组。

注意：

答案中不可以包含重复的四元组。

示例：

``` text
给定数组 nums = [1, 0, -1, 0, -2, 2]，和 target = 0。

满足要求的四元组集合为：
[
  [-1,  0, 0, 1],
  [-2, -1, 1, 2],
  [-2,  0, 0, 2]
]
```

### 解答

跟三数之和逻辑一样，只需多加一层遍历即可

``` java
public static List<List<Integer>> solution(int[] nums, int target) {
    // 数量不足返回空
    if (nums == null || nums.length < 4) {
        return Collections.emptyList();
    }
    List<List<Integer>> ans = new ArrayList<>();
    Arrays.sort(nums);
    int len = nums.length;
    // 外层循环
    for (int i = 0; i < len - 3; i++) {
        // 如果数值已经判断过了，直接跳过，去重
        if (i > 0 && nums[i] == nums[i - 1]) {
            continue;
        }
        // 如果当前的最小值比target还小，继续循环是找不到的，break
        int min_i = nums[i] + nums[i + 1] + nums[i + 2] + nums[i + 3];
        if (target < min_i) {
            break;
        }
        // 当前的最大值比target还打，继续循环是有可能找到，continue
        int max_i = nums[i] + nums[len - 1] + nums[len - 2] + nums[len - 3];
        if (target > max_i) {
            continue;
        }
        // 内层循环
        for (int k = i + 1; k < len - 2; k++) {
            // 已经判断过则跳过，去重
            if (k > i + 1 && nums[k] == nums[k - 1]) {
                continue;
            }
            int left = k + 1;
            int right = len - 1;
            // 跟外层一样的判断逻辑
            int min_k = nums[i] + nums[k] + nums[k + 1] + nums[k + 2];
            if (target < min_k) {
                break;
            }
            int max_k = nums[i] + nums[k] + nums[len - 1] + nums[len - 2];
            if (target > max_k) {
                continue;
            }
            // 双指针向中间遍历
            while (left < right) {
                int sum = nums[i] + nums[k] + nums[left] + nums[right];
                if (target == sum) {
                    ans.add(Arrays.asList(nums[i], nums[k], nums[left], nums[right]));
                    // 去重的逻辑:当找到一组时，判断其附近是否有相等的值
                    while (left < right && nums[left] == nums[left + 1]) {
                        left++;
                    }
                    while (left < right && nums[right] == nums[right - 1]) {
                        right--;
                    }
                }
                if (target < sum) {
                    right--;
                } else {
                    left++;
                }
            }
        }
    }
    return ans;
}
```
