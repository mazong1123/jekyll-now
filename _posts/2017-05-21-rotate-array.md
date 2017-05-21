---
layout: post
title: Rotate array
date: 2017-05-21 22:51
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# How to rotate an array clock-wise in kth position?

For example, with n = 7 and k = 3, the array [1,2,3,4,5,6,7] is rotated to [5,6,7,1,2,3,4].

With n = 7 and k = 2, the array [1,2,3,4,5,6,7] is rottated to [6,7,1,2,3,4,5].

LeetCode link: https://leetcode.com/problems/rotate-array

```cpp
class Solution {
public:
    void rotate(vector<int>& nums, int k) {
        if (k <= 0)
        {
            return;
        }

        k = k % nums.size();

        reverse(nums.begin(), nums.begin() + nums.size() - k);
        reverse(nums.begin() + nums.size() - k, nums.end());
        reverse(nums.begin(), nums.end());
    }
};
```
