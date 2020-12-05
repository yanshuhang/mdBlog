# 旋转图像

## 问题描述

给定一个 n × n 的二维矩阵表示一个图像。

将图像顺时针旋转 90 度。

## 解答

交换同一个斜线上的,然后反转每一层数组

``` java
public void solution(int[][] matrix) {
    int len = matrix.length;
    // 交换同一个斜线上的元素
    for (int i = 0; i < len - 1; i++) {
        for (int j = i+1; j < len; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }
    // 再反转每一层
    for (int i = 0; i < len; i++) {
        int left = 0;
        int right = len - 1;
        while (left < right) {
            int temp = matrix[i][right];
            matrix[i][right] = matrix[i][left];
            matrix[i][left] = temp;
            left++;
            right--;
        }
    }
}
```
