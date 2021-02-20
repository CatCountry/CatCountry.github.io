---
layout: psot
title: leetcode-1-两数之和
date: 2021-02-20 16:51:24
tags: ['leetcode', 'N数之和']
categories: leetcode
---

## 两数之和

### 题目

给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 的那 两个 整数，并返回它们的数组下标。 
可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。 
可以按任意顺序返回答案。 

例子
```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
```

限制
2 <= nums.length <= 103 
-109 <= nums[i] <= 109 
-109 <= target <= 109 
只会存在一个有效答案 

### 解法
#### 1. 暴力建立
暴力遍历是最容易想到的方案，整体时间复杂度是O（n^2）。
代码：
```Java
    public int[] twoSum(int[] nums, int target) {
        int[] results = new int[2];
        for (int i = 0; i < nums.length; i++) {
            int j = i + 1;
            while (j < nums.length) {
                if (nums[i] + nums[j] == target) {
                    results[0] = i;
                    results[1] = j;
                    return results;
                }
                j ++;
            }
        }
        return results;
    }
```

#### 2. 哈希映射
将数组的值和index建立映射(时间复杂度为O(n))，然后遍历数组寻找`target - arr[i]`只需要O(1)的时间复杂度就可以找到是否存在另外的一个元素.
所以整体的时间复杂度为O(n);

代码：
```Java
    public int[] twoSum(int[] nums, int target) {
        int[] results = new int[2];
        Map<Integer, Integer> valueIndexMap = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            valueIndexMap.put(nums[i], i);
        }
        for (int i = 0; i < nums.length; i++) {
            Integer value = valueIndexMap.get(target - nums[i]);
            if (value != null && value != i) {
                results[0] = i;
                results[1] = value;
                return results;
            }
        }
        return results;
    }
```