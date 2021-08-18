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



# 动态规划

## 5、最长回文子串
给你一个字符串 s，找到 s 中最长的回文子串。

示例 1：

输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
示例 2：

输入：s = "cbbd"
输出："bb"

链接：https://leetcode-cn.com/problems/longest-palindromic-substring

思路：
考虑用动态规划，假设s的长度为len, 
d[i]表示字符串前i个字符的最长回文字符长度，可是这种方法只能得到最长回文子串的长度，不能得到最长回文子串本身。
if((i+1%2) ==0) d[i+1] = d[i];//如果i前面有奇数个，那么加上s[i+1]也不会增加回文数。比如前面是aba，无论再加什么回文数也不增加
else if(s[i+1] ==s[0])//思路错误，如果i前面有奇数给，增加s[i+1]是可能增加回文数的，比如前面是cba,如果再来个b就可以增加回文数。

换个思路：d[i]表示字符串前i个字符的最长回文字符长度，假如回文字符长度为m,那么假如再增加一个字符，即s[i+1],会有什么可能性呢？

d[0] = 0；
d[1] = 1；
d[2] = 1;

abc  abcd  m
ab    aba
a     ac
bac   baca
abcb   abcba

if(s[i+1] == s[i-d[i]-1]) d[i+1] = d[i]+1 //根据以上观察简单列出
else d[i+1] =d[i]
//if(s[i] == s[i-d[i-1]-1]) d[i+1] = d[i]+1 //根据以上观察简单列出
d[i] = m;
d[i+1] =?

for(int i = 2; i< s.length; i++){
    if(s[i+1] == s[i-d[i]-1]) d[i+1] = d[i]+1 ;
    else d[i+1] = d[i]
}


```
class Solution {
public:
    string longestPalindrome(string s) {
        string str;
        int len = s.length();
        vector<int> dp(len, 0);
        dp[1] = 1;

        for(int i = 2; i<len; i++){
            cout<<i<<endl;
            if(s[i] == s[i-dp[i-1]-1]){
                dp[i] = dp[i-1]+1;
                cout<<"dp "<<dp[i]<<endl;
            }else{
                dp[i] = dp[i-1];
                cout<<"dp "<<dp[i]<<endl;
            }
        }
        int index;
        for(int i = 0; i< len; i++){
            if(dp[i] == dp[len-1]){
                index = i;
            }
        }
        for(int j = index-dp[len-1]+1; j<=index; j++){
            str.append(s[j]);
        }

        return str;
    }
};
```

事实证明，上面的方法根本不对，思路就错了，下面先考虑一下暴力破解的思路:
- 找到所有子串
- 判断是否是回文子串
- 如果是回文子串且长度大于之前的，记录下来。
  
```
class Solution {
public:
    bool isPalindrome(string s){

        int len = s.length();
        for (int i = 0; i < len/2; i++)
        {
            if (s[i] != s[len-i-1])
            {
                return false;
            }
            
        }
        return true;
    }

    string longestPalindrome(string s) {
        string ans = "";
        int max = 0;
        int length = s.length();
        if (length < 2)
        {
            return s;
        }
        
        for (int  i = 0; i < length; i++) {
            for (int j = 0; j <= length-i; j++) {
                string temp = s.substr(i,j);//s.substr(pos, n)
                if (isPalindrome(temp)&& temp.length() > max)
                {
                    ans = temp;
                    max = temp.length();
                }
            }
        }
    return ans;
    }
};
```
在上面的代码中，我用到了substring，由于Java和C++的差异，让我折腾了好久。
Java中：
`public String substring(int beginIndex, int endIndex)`
C++中：
` s.substr(pos, len)`
可以看到Java中，参数分别为首和尾+1，而C++中，参数分别为首和取子串长度。
差异还是比较大的。结果我弄混淆了。