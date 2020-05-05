---
title: leetcode - 3、回文数
date: 2019-11-14 18:00:04
tags:
    - leetcode
categories:
    - leetcode
---

### 题目描述
判断一个整数是否是回文数。回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

示例 1:

> 输入: 121
> 输出: true

示例 2:

> 输入: -121
> 输出: false
> 解释: 从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。

示例 3:

> 输入: 10
> 输出: false
> 解释: 从右向左读, 为 01 。因此它不是一个回文数。

进阶:

> 你能不将整数转为字符串来解决这个问题吗？

### 解
#### Round 1
看到进阶的说明慧心一笑，嘿嘿。这不就告诉我们可以通过字符串反转来对比。
```java
class Solution {
    public boolean isPalindrome(int x) {
        String a = String.valueOf(x);
        String b = new StringBuffer(a).reverse().toString();
        if(a.equals(b)) {
            return true;
        }
        return false;
    }
}
```
