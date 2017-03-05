---
layout: post
title: Assign cookie solution and explanation
date: 2017-03-05 22:42
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Problem Description

Below is a description of the assign cookie:


Assume you are an awesome parent and want to give your children some cookies. But, you should give each child at most one cookie. Each child i has a greed factor gi, which is the minimum size of a cookie that the child will be content with; and each cookie j has a size sj. If sj >= gi, we can assign the cookie j to the child i, and the child i will be content. Your goal is to maximize the number of your content children and output the maximum number.

Note:
You may assume the greed factor is always positive.
You cannot assign more than one cookie to one child.

Example 1:
Input: [1,2,3], [1,1]

Output: 1

Explanation: You have 3 children and 2 cookies. The greed factors of 3 children are 1, 2, 3.
And even though you have 2 cookies, since their size is both 1, you could only make the child whose greed factor is 1 content.
You need to output 1.
Example 2:
Input: [1,2], [1,2,3]

Output: 2

Explanation: You have 2 children and 3 cookies. The greed factors of 2 children are 1, 2.
You have 3 cookies and their sizes are big enough to gratify all of the children,
You need to output 2.


The problem can be found on [LeetCode](https://leetcode.com/problems/assign-cookies)

# Analysis

The question description can be translated as following: Find the numbers from the 2nd array which are greater than or equal to the numbers in the 1st array as many as possible. We have a constrain that the numbers in the second array can only be used one time.

The naive method is obvious: Just pick every element in the 1st array, find if any number greater than or equal to it in the 2nd array. If found, remove that number from the 2nd array. So we have nested loops in this method. The outer loop intends to pick every element in the 1st array, whereas the inner loop tries to find a number in the 2nd array which is greater than or equal to the picked number in the 1st array. The time complexity is O(n^2) - SLOW!!!

Can we do better?

You'll notice that this problem is all about "Find and comparison". How can we speed up find and comparison? Yes, it's sorting. If the 2nd array is **sorted**, we'll have an important property:

#### If the last number in the 2nd array is less than the target number in the 1st array, no number in the 2nd array can match the target number.

Why? Because the last number in the 2nd array is the biggest number, and it is less than the target number. Then no number in the 2nd number can be greater than or equal to the target number.

This property makes the comparison time complexity O(1). Consider following cases:

1. If the last number in the 2nd array is >= target number, we found the matched number, done.
2. If the last number in the 2nd array is < target number, no matched number in the 2nd array can be found, done.

Obviously, the inner loop can be eliminated. We just need exactly 1 comparison which is O(1)

Are we done? Nope. Consider following case:

[2, 7, 5] [3, 9]

9 is > 2, so we find a match number. However, 3 is less than 7 and 5, we will get a wrong answer. In fact, because 9 > 7 and 3 > 2, we should output 2.

What's the problem? Because we're "GREEDY". We used 9 to match 2, that's a waste. If we check the **BIGGEST** number in the 1st array at first, and the 2nd big number, so forth, we'll get the right answer.

Now you'll aware that we should sort the 1st array as well. Then we can loop the 1st array from last element to the first element, check if any matched number exists in the 2nd array.

How about the time complexity? It can be easily calculated:

```
T = sorttime(A1) + sorttime(A2) + O(n * 1)
```

We're gonna use quick-sort here, so we can get the average time complexity:

```
T = 2 * O(nlgn) + O(n * 1) = O(nlgn)
```

Now let's write code:

```cpp
class Solution {
public:
    int findContentChildren(vector<int>& g, vector<int>& s) {
        if (g.size() == 0 || s.size() == 0)
        {
            return 0;
        }

        sort(g.begin(), g.end());
        sort(s.begin(), s.end());

        int i = g.size() - 1;
        int j = s.size() - 1;

        int r = 0;
        while (i >= 0 && j >= 0)
        {
            if (s[j] >= g[i])
            {
                ++r;

                --j;
                --i;
            }
            else
            {
                --i;
            }
        }

        return r;
    }
};
```

Happy coding :)
