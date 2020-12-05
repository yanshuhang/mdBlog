# N皇后II

## 问题描述

给定一个整数 n，返回 n 皇后不同的解决方案的数量。

## 解答

不需要记录位置了, 只在成功之后返回1加到count中

``` java
public int solution(int n) {
    boolean[] columns = new boolean[n];
    boolean[] diagonals1 = new boolean[2 * n - 1];
    boolean[] diagonals2 = new boolean[2 * n - 1];
    return backtrack(columns, diagonals1, diagonals2, n, 0);
}

public int backtrack(boolean[] columns, boolean[] diagonals1, boolean[] diagonals2, int n, int row) {
    if (row == n) {
        return 1;
    } else {
        int count = 0;
        for (int column = 0; column < n; column++) {
            int diagonal1 = row + column;
            int diagonal2 = n - 1 + (row - column);
            if (columns[column] || diagonals1[diagonal1] || diagonals2[diagonal2]) {
                continue;
            }
            columns[column] = diagonals1[diagonal1] = diagonals2[diagonal2] = true;
            count += backtrack(columns, diagonals1, diagonals2, n, row + 1);
            columns[column] = diagonals1[diagonal1] = diagonals2[diagonal2] = false;
        }
        return count;
    }
}
```
