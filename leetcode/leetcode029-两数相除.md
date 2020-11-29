# 两数相除

## 问题描述

给定两个整数，被除数 dividend 和除数 divisor。将两数相除，要求不使用乘法、除法和 mod 运算符。

返回被除数 dividend 除以除数 divisor 得到的商。

整数除法的结果应当截去（truncate）其小数部分，例如：truncate(8.345) = 8 以及 truncate(-2.7335) = -2

## 解答

主要思想是通过减法来求解除法，如果一次循环减一次除数，循环次数会过多，其中有几个可优化的地方：

* 每次剪掉除数后，将除数翻倍，结果也翻倍记录，然后再循环直到被除数小于除数
* 两个数都变为负数进行处理，这样不会溢出

``` java
public static int solution(int dividend, int divisor) {
    // 0除以任何数都是0
    if (dividend == 0) {
        return 0;
    }
    // 任何数除以1都是自身
    if (divisor == 1) {
        return dividend;
    }
    // 可能会溢出的情况，取最大值
    if (dividend == Integer.MIN_VALUE && divisor == -1) {
        return Integer.MAX_VALUE;
    }
    // 保存结果的正负
    boolean flag = (dividend > 0 && divisor < 0) || (dividend < 0 && divisor > 0);
    // 转换为负数处理，防溢出
    dividend = -Math.abs(dividend);
    divisor = -Math.abs(divisor);
    int result = 0;
    while (dividend <= divisor) {
        int timesDivisor = divisor;
        int times = 1;
        // 每次翻倍可以省去循环次数
        while (dividend - timesDivisor <= timesDivisor) {
            timesDivisor += timesDivisor;
            times += times;
        }
        // 记录下结果,继续外部的循环
        dividend -= timesDivisor;
        result += times;
    }
    return flag ? -result : result;
}
```
