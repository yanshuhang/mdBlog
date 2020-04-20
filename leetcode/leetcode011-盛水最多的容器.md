# 盛水最多的容器

## 问题描述

给你 n 个非负整数 a1，a2，...，an，每个数代表坐标中的一个点 (i, ai) 。在坐标内画 n 条垂直线，垂直线 i 的两个端点分别为 (i, ai) 和 (i, 0)。找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

说明：你不能倾斜容器，且 n 的值至少为 2。

![图片](..\pic\question_11.jpg)
图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。

示例：

``` text
输入：[1,8,6,2,5,4,8,3,7]
输出：49
```

## 解答

* 暴力法：遍历所有可能，找出最大值

``` java
public static int solution(int[] height) {
    int result = 0;
    for (int i = 0; i < height.length - 1; i++) {
        for (int j = i + 1; j < height.length; j++) {
            result = Math.max(result, Math.min(height[i], height[j]) * (j - i));
        }
    }
    return result;
}
```

* 从两边往中间求解：由于容器面积取决与两边最小的一边，此时向中间遍历有两种：

1. 较小边不动，从较大边向中间遍历，面积只会变小

2. 较大边不动， 从较小边向中间遍历，如果较小边变大则有可能面积变大

所以我们比较出大小边之后，采取情况2，较小边向中间遍历，直到两边相遇。

``` java
public static int solution1(int[] height) {
    int result = 0;
    // 记录下最小边，即高度
    int h = 0;
    // 当前面积
    int v = 0;
    for (int i = 0, j = height.length - 1; i < j; ) {
        // 取出较小的数值，然后其索引移动
        h = height[i] < height[j] ? height[i++] : height[j--];
        v = h * (j - i + 1);
        result = Math.max(result, v);
    }
    return result;
}
```

* 在上一种解法中再优化：如果索引移动之后的值不大于之前的值，那面积肯定是比之前小的，不需要计算了。这个提交之后确实快了1ms。

``` java
public static int solution2(int[] height) {
    int result = 0;
    int h = 0;
    for (int i = 0, j = height.length - 1; i < j;) {
        if (height[i] < height[j]) {
            h = height[i];
            result = Math.max(result, (j - i) * h);
            while (i < j && height[++i] < h) {
                i++;
            }
        } else {
            h = height[j];
            result = Math.max(result, (j - i) * h);
            while (i < j && height[--j] < h) {
                j--;
            }
        }
    }
    return result;
}
```
