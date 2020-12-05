# N皇后

## 问题描述

n 皇后问题研究的是如何将 n 个皇后放置在 n×n 的棋盘上，并且使皇后彼此之间不能相互攻击。  
给定一个整数 n，返回所有不同的 n 皇后问题的解决方案。  
皇后彼此不能相互攻击，也就是说：任何两个皇后都不能处于同一条横行、纵行或斜线上。

## 解答

回溯法: 使用三个数组记录纵行和两个斜线上是否已经有了皇后

1. 尝试在第一层第一个位置放置一个皇后
2. 如果三个数组记录中没有皇后,那么在下一层尝试放置皇后
3. 如果三个数组记录中已有皇后,继续下一个位置,如果所有位置都不成功,说明上一层的不对
4. 回到上一层,尝试下一个位置
5. 直到最后一层放置成功,将结果添加到最终结果集中

主要是斜线上的规律总结:

1. 正斜线上的其`row+clumn`相等
2. 反斜线上的其`row-clumn`相等

``` java
public List<List<String>> solution1(int n) {
    List<List<String>> ans = new ArrayList<>();
    // 初始化棋盘
    char[][] board = new char[n][n];
    for (char[] chars : board) {
        Arrays.fill(chars, '.');
    }
    // 记录皇后是否冲突
    boolean[] colums = new boolean[n];
    boolean[] diagonals1 = new boolean[2 * n - 1];
    boolean[] diagonals2 = new boolean[2 * n - 1];
    // 回溯
    backtrack1(ans, board, colums, diagonals1, diagonals2, n, 0);
    return ans;
}

public void backtrack1(List<List<String>> ans, char[][] board, boolean[] colums, boolean[] diagonals1, boolean[] diagonals2, int n, int row) {
    // 成功
    if (row == n) {
        List<String> list = new ArrayList<>();
        for (char[] chars : board) {
            list.add(new String(chars));
        }
        ans.add(list);
    } else {
        // 遍历本层位置,尝试放置皇后
        for (int i = 0; i < n; i++) {
            // 计算斜线对应的索引
            int diagonal1 = n - 1 + (row - i);
            int diagonal2 = row + i;
            // 判断是否冲突
            if (colums[i] || diagonals1[diagonal1] || diagonals2[diagonal2]) {
                continue;
            }
            // 不冲突则放置皇后并修改状态
            board[i][row] = 'Q';
            colums[i] =diagonals1[diagonal1] = diagonals2[diagonal2] = true;
            // 进入下一层尝试放置
            backtrack1(ans, board, colums, diagonals1, diagonals2, n, row + 1);
            // 回溯后剪枝
            colums[i] = diagonals1[diagonal1] = diagonals2[diagonal2] = false;
            board[i][row] = '.';
        }
    }
}
```
