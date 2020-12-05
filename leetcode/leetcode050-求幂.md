# 求幂

## 问题描述

实现 pow(x, n) ，即计算 x 的 n 次幂函数。

## 解答

最简单的是将x乘n次,相当于使用1+1+1+...直到n, 问题就变成了怎么更快的得到n  
最快的算到n的方法是:二分,即将n换算成其二进制的表示,每次与自身相加是最快的算到n的方法

1. 遇到0时将x平方,相当于n的二进制进了一位
2. 遇到1时,将x乘到最终结果中,然后再将x平方

``` java
public double solution(double x, int n) {
    if (x == 0.0) {
        return 0.0;
    }
    long N = n;
    return N >= 0 ? quickMul(x, N) : 1.0 / quickMul(x, -N);
}
public double quickMul(double x, long n) {
    double ans = 1.0;
    double temp = x;
    while (n > 0) {
        if (n % 2 == 1) {
            ans *= temp;
        }
        temp *= temp;
        n /= 2;
    }
    return ans;
}
```
