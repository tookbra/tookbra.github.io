---
title: leetcode - 1、two sum
date: 2019-01-07 22:28:23
tags:
    - leetcode
categories:
    - leetcode
mathjax: true
---

![two-sum](/images/leetcode/two-sum.png)

### 题目描述
给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:

>    给定 nums = [2, 7, 11, 15], target = 9
>    因为 nums[0] + nums[1] = 2 + 7 = 9
>    所以返回 [0, 1]

<!-- more -->
### 解
#### Round 1
撸起袖子一把梭，2层for循环暴力破解，时间复杂度$O(n^2)$

```java
class Solution {

    public int[] twoSum(int[] nums, int target) {
        int [] result = new int[2];
        for(int i = 0; i < nums.length; i++) {
            for(int k = i + 1; k < nums.length; k++) {
                if(nums[i] + nums[k] == target) {
                    return new int[] {i, k};
                }
            }
        }
    }
}
```

#### Round 2
有没有可能把时间复杂度降到$O(n)$呢，我们有个目标值target来看看这里面能不能做文章;
既然2数之和等于target的值，那我们是不是用target减去其中一个数就是另外一个数了呢,再去数组里找对应的这个数，时间复杂度不就变成$O(n)$了?
$num1 + num2 = target$ -> $num2 = target- num1$

```java
class Solution {

    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap();
        for(int i = 0; i < nums.length; i++) {
            int num2 = target - nums[i];
            if(map.containsKey(num2) && map.get(num2) != i) {
                return new int[] {i, map.get(num2)};
            }
            map.put(nums[i], i);
        }
        return null;
    }
}
```

