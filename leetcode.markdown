---
layout: page
title: LeetCode
permalink: /LeetCode/
---

# LeetCode-1-Two-Sum
给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。
你可以按任意顺序返回答案。



Example 1:

Input: nums = [2,7,11,15], target = 9
Output: [0,1]
Output: Because nums[0] + nums[1] == 9, we return [0, 1].

解法1：
```
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        vector<int> arr;
        for(int i = 0; i<nums.size()-1;i++){
            for(int j = i+1; j < nums.size(); j++){
                if(nums[i]+nums[j] == target){
                    arr.push_back(i);
                    arr.push_back(j);
                    return arr;
                }
            }
        }

        return arr;
    }
};

```


# LeetCode-2-两数相加
给你两个 非空 的链表，表示两个非负的整数。它们每位数字都是按照 逆序 的方式存储的，并且每个节点只能存储 一位 数字。

请你将两个数相加，并以相同形式返回一个表示和的链表。

你可以假设除了数字 0 之外，这两个数都不会以 0 开头。
[add two numbers](https://leetcode-cn.com/problems/add-two-numbers)

![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2021/01/02/addtwonumber1.jpg)

```
输入：l1 = [2,4,3], l2 = [5,6,4]
输出：[7,0,8]
解释：342 + 465 = 807.
```



