---
title: leetcode - 2、add tow numbers
date: 2019-01-08 17:46:33
img: /images/leetcode/two-sum.png
tags:
    - leetcode
categories:
    - leetcode
mathjax: true
---

![two-sum](/images/leetcode/two-sum.png)

### 题目描述
给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。

示例1:
>    输入: 123
>    输出: 321

示例2:

>    输入: -123
>    输出: -321
示例3:


>    输入: 120
>    输出: 21
>
注意:
假设我们的环境只能存储得下 32 位的有符号整数，则其数值范围为 $[−2^{31},  2^{31}-1]$。请根据这个假设，如果反转后整数溢出那么就返回 0。     
<!-- more -->

### 解
#### Round 1
```java
class Solution {
    public int reverse(int x) {
        if(x == 0) {
            return x;
        }
        boolean flag = false;
        long temp = x;
        if(temp < 0) {
            flag = true;
            temp = -temp;
        }
        StringBuffer sb = new StringBuffer(String.valueOf(temp));
        sb.reverse();
        Long result = Long.valueOf(sb.toString());
        if(result > Integer.MAX_VALUE) {
            return 0;
        }
        if(result < Integer.MIN_VALUE) {
            return 0;
        }
        return flag ? -(result.intValue()) : result.intValue();
    }
}
```
#### Round 2
```java
class Solution {
    public int reverse(int x) {
        long result = 0;
        while (x != 0) {
            result = result * 10 + (x % 10);
            x /= 10;
        }
        if(result < Integer.MIN_VALUE || result > Integer.MAX_VALUE) {
            return 0;
        }
        return (int) result;
    }
}
```
