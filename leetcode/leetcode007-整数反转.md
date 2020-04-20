# 整数反转

## 问题描述

给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。

示例 1:

``` text
输入: 123
输出: 321
```

 示例 2:

``` text
输入: -123
输出: -321
```

示例 3:

``` text
输入: 120
输出: 21
```

注意:

假设我们的环境只能存储得下 32 位的有符号整数，请根据这个假设，如果反转后整数溢出那么就返回 0。

## 解答

还是很简单的，通过10的余数把每一位求出来加到result*10上即可，主要是判断是否超出了32位，如果result可以使用long类型，判断会更简单一些。  
java的求余保留了符号，所有符号不需要进行判断了。

``` java
public int reverse(int x) {
        int result = 0;

        do {
            int num = x % 10;
            if (result > Integer.MAX_VALUE/10 || (result == Integer.MAX_VALUE / 10 && num > 7)) return 0;
            if (result < Integer.MIN_VALUE/10 || (result == Integer.MIN_VALUE / 10 && num < -8)) return 0;
            result = result * 10 + num;
        } while ((x = x /10) != 0);

        return result;

    }
```
