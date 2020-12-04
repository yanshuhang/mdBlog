# 接雨水

## 问题描述

## 解答

* 暴力法:计算每个位置处能存储的雨水,加在一起就是总的雨水量  
    计算单个位置的雨水量等于其两边最大高度的较小值减去当前的高度  
    需要遍历数组一遍,每次遍历又需要向左右遍历查找最大值

``` java
public int solution(int[] height) {
    if (height == null || height.length == 0) {
        return 0;
    }
    int ans = 0;
    int size = height.length;
    for (int i = 0; i < size - 1; i++) {
        int max_left = 0, max_right = 0;
        // 找出当前位置两边的最大值
        for (int j = i; j < size; j++) {
            max_right = Math.max(max_right, height[j]);
        }
        for (int j = i; j >= 0; j--) {
            max_left = Math.max(max_left, height[j]);
        }
        // 其中的较小的减去当前高度既是当前位置的雨水量
        ans += Math.min(max_left, max_right) - height[i];
    }
    return ans;
}
```

* 在暴力法中查找当前位置两边的最大值的过程中有很多重复的比较,我们可以两次遍历使用两个数组记录下所有位置的两边最大值.其余的解法跟暴力法一样

``` java
public int solution1(int[] height) {
    if (height == null || height.length == 0) {
        return 0;
    }
    int ans = 0;
    int size = height.length;
    // 存储每个位置的两边最大值
    int[] left_max = new int[size];
    int[] right_max = new int[size];
    left_max[0] = height[0];
    // 两个for循环记录下每个位置的两边最大值
    for (int i = 1; i < size; i++) {
        left_max[i] = Math.max(height[i], left_max[i - 1]);
    }
    right_max[size - 1] = height[size - 1];
    for (int i = size - 2; i >= 0; i--) {
        right_max[i] = Math.max(height[i], right_max[i + 1]);
    }
    // 最后一个for循环遍历求得每个位置的雨水量,并加起来
    for (int i = 1; i < size - 1; i++) {
        ans += Math.min(left_max[i], right_max[i]) - height[i];
    }
    return ans;
}
```

* 双指针法: left和right两个指针从左右遍历

  * 在遍历时记录下left指针左侧的最大值left_max, right指针右侧的最大值right_max
  * 如果height[left] 小于 height[right]
    * left_max 小于等于 left处的值,left处放不了雨水,更新left_max的值
    * left_max 大于 left处的值,计算方法跟暴力法一样
    * 然后left指针右移
  * 相反时,right指针的判断也是一样的逻辑

``` java
public int solution2(int[] height) {
    if (height == null || height.length == 0) {
        return 0;
    }
    int ans = 0;
    // 双指针及其对于的最大值
    int left = 0, right = height.length - 1;
    int left_max = 0, right_max = 0;
    while (left < right) {
        int l_val = height[left];
        int r_val = height[right];
        // 左右指针处的值哪个小,先判断哪个
        if (l_val < r_val) {
            if (left_max <= l_val) {
                left_max = l_val;
            } else {
                ans += (left_max - l_val);
            }
            left++;
        } else {
            if (right_max <= r_val) {
                right_max = r_val;
            } else {
                ans += (right_max - r_val);
            }
            right--;
        }
    }
    return ans;
}
```
