# 跳跃游戏II

## 问题描述

## 解答

* 反向查找:从最后一个位置开始,每次都查找可以跳到该位置的最远的索引,直至第一个位置  
    但是对于每个位置都要遍历一遍,太慢了

``` java
public int solution(int[] nums) {
    int index = nums.length - 1;
    int count = 0;
    while (index > 0) {
        for (int i = 0; i < index; i++) {
            // 从左边遍历找到第一个可以跳到index的位置,就是最远的
            // 修改index, count++, 中断for循环继续while循环
            if (i + nums[i] >= index) {
                index = i;
                count++;
                break;
            }
        }
    }
    return count;
}
```

* 只遍历一遍,维护一个遍历position表示当前步可以跳到的最大位置  
    1. 首先记录下当前可跳到的最大位置

    2. 遍历当前到最大位置,记录下再跳一步的最大位置

    3. 遍历到当前最大位置时,step + 1 ,相当于一步跳到了该位置

``` java
public int solution1(int[] nums) {
    int position = 0;
    int step = 0;
    int end = 0;
    for (int i = 0; i < nums.length - 1; i++) {
        // position记录下每一步可以跳到的最大位置
        // 然后在求得当前步可以达到的所有位置的下一步跳到的位置
        // 得到下一步的最大位置
        // 每当遍历达到该位置step+1
        position = Math.max(position, i + nums[i]);
        if (i == end) {
            end = position;
            step++;
        }
    }
    return step;
}
```
