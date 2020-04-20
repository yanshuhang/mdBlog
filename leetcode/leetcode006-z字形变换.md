# z字形变换

## 问题描述

将一个给定字符串根据给定的行数，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 "LEETCODEISHIRING" 行数为 3 时，排列如下：

``` text
L   C   I   R
E T O E S I I G
E   D   H   N
```

之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："LCIRETOESIIGEDHN"。

请你实现这个将字符串进行指定行数变换的函数：

``` text
string convert(string s, int numRows);
```

示例 1:

``` text
输入: s = "LEETCODEISHIRING", numRows = 3
输出: "LCIRETOESIIGEDHN"
```

示例 2:

``` text
输入: s = "LEETCODEISHIRING", numRows = 4
输出: "LDREOEIIECIHNTSG"
解释:

L     D     R
E   O E   I I
E C   I H   N
T     S     G
```

## 解答

创建多个stringBuilder对应多行，遍历字符串，从第1行开始append字符，然后下一行append，当走到最后一行时开始往上走，走到第1行时再往下走

``` java
public String solution(String s, int numRows) {
    if (numRows == 1 || numRows >= s.length()) {
        return s;
    }
    // 每一行对应一个StringBuilder
    List<StringBuilder> rows = new ArrayList<>(numRows);
    for (int i = 0; i < rows.size(); i++) {
        rows.add(new StringBuilder());
    }

    int curRow = 0;
    boolean goingDown = false;

    for (char c : s.toCharArray()) {
        rows.get(curRow).append(c);

        // 每当最后一行append之后curRow-1，然后到达第一行开始curRow+1
        if (curRow == 0 || curRow == numRows - 1) {
            goingDown = !goingDown;
        }
        curRow += goingDown ? 1 : -1;
    }

    StringBuilder rt = new StringBuilder();
    for (StringBuilder row : rows) {
        rt.append(row);
    }
    return rt.toString();
}
```
