# 有效的数独

## 问题描述

判断一个 9x9 的数独是否有效。只需要根据以下规则，验证已经填入的数字是否有效即可。

* 数字 1-9 在每一行只能出现一次。
* 数字 1-9 在每一列只能出现一次。
* 数字 1-9 在每一个以粗实线分隔的 3x3 宫内只能出现一次。

数独部分空格内已填入了数字，空白格用 '.' 表示。

## 解答

使用3个二维数组存储数字在数独中是否已存在

* 3个二维数组分别对于行,列和3*3宫格
* 遍历一遍,如果有已存在即返回false,否则将信息填入二维数组中

``` java
public static boolean solution(char[][] board) {
    boolean[][] rows = new boolean[9][9];
    boolean[][] colums = new boolean[9][9];
    boolean[][] boxes = new boolean[9][9];
    for (int i = 0; i < 9; i++) {
        for (int j = 0; j < 9; j++) {
            int value = board[i][j] - '1';
            int boxIndex = (i / 3) + (j / 3) * 3;

            if (rows[i][value] || colums[j][value] || boxes[boxIndex][value]) {
                return false;
            }
            rows[i][value] = true;
            colums[i][value] = true;
            boxes[i][value] = true;
        }
    }
    return true;
}
```
