---
layout: post
title: Minimum moves to equal array solution and explanation
date: 2017-03-11 20:16
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Problem Description

Below is a description of the problem:


Given a non-empty integer array of size n, find the minimum number of moves required to make all array elements equal, where a move is incrementing n - 1 elements by 1.

Example:

Input:
[1,2,3]

Output:
3

Explanation:
Only three moves are needed (remember each move increments two elements):

[1,2,3]  =>  [2,3,3]  =>  [3,4,3]  =>  [4,4,4]

The problem can be found on [LeetCode](https://leetcode.com/problems/minimum-moves-to-equal-array-elements)

# Analysis

The problem can be translated to:

How to eliminate the differences of the numbers in an array in minimum steps. In each step, you can keep 1 element inert, and increase the value of other elements in the array".

The intuition tells us the number of steps is dominated by the minimum and maximum numbers in the array. Because the minimum number has the largest differences to the maximum number.

So the strategy is simple:

1. Find the maximum number in the array.
2. Increase the numbers other than the maximum number.
3. Check if all numbers in the array are equal.
4. If all numbers are equal, we're done. Otherwise, go to step 1.

How about the time complexity? Let's say we have n elements in the array. We have to iterate the array to find the maximum number so we'll take O(n) in step 1. In step 2, we still need to iterate the array so we have O(n) either. Needless to say, we should iterate the array in step 3, so we take O(n) as well. Now step 4 introduces a loop for step 1, 2 and 3 and the loop count is the minimum moves m. The final time complexity is O(m * 3n).

If the differences between minimum and maximum number large enough, we may suffer a performance problem. For example, if we have following n size array:

[1, 1, ..., 1, 2147483647]

The m is big enough to make the O(m * 3n) time complexity slow. Even if we can easily calculate the minimum moves (2147483647 - 1 = 2147483646), our method still needs to waste loops and checks to get the result.

OK, can we do better?

Take a look at the steps in our method. We depends on the iteration heavily. We cannot improve the performance if the iterations cannot be eliminated.

Let's look at the final state of the array: we have n elements with a same value f. What's the changes to the initial status? In each move, we have to increase the values of n - 1 elements. So when we reach the final status, we increased m * (n - 1) values to the initial array. Now we get a very important property:

```
sum(array) + m * (n - 1) = n * f
```

Our task is to calculate m. So we have:

```
m = (n * f - sum(array)) / (n - 1)
```

Now we have a new problem: We knew n, Sum(array) but not f. How can we solve this equation?

Let's think reverse. If we knew f, we can solve the problem. We knew all information of the input array, how can we calculate the f?

Ultimately, each element in the array will be increased to f after m moves. Let's say s[i] is the increased value of the ith element after m moves to reach the value f. We have following equation:

```
a[i] + s[i] = f
```

We knew a[i], now we should calculate s[i]. It's trivial to prove 0 <= s[i] <= m. Because each move may increase 1 of a[i], the maximum increased value is m * 1 = m, the minimum increased value is 0 (e.g: [1, 2], no need to increase the second element so s[1] == 0), that is 0 <= s[i] <= m.

How to make s[i] == m ? That means in each move, a[i] should be increased. Which element should be treated like this? First, we can pass over the maximum number. Because it is has biggest chance to avoid being increased. We cannot guarantee other elements to be increased in each move except the minimum element. The minimum element, has no chance to be the maximum number during the movement, because we only increase 1 for the elements other than the maximum element. Other elements have the chance to be the maximum elements during the movement so they may not require a incremental in its turn (e.g: [1,2,3] => [2,3,3] => [3,3,4] => [4,4,4], s[0] == 3, s[1] == 2, s[2] == 1). Now we get:

```
min(array) + m = f
```

Recall we have:

```
m = (n * f - sum(array)) / (n - 1)
```

Now we get:

```
m = (n * (min(array) + m) - sum(array)) / (n - 1)
```

Solve the equation we get:

```
m = sum(array) - n * min(array)
```

Let's calculate the time complexity. We just need 1 loop to calculate the sum of array and find the minimum number in the array so the time complexity is O(n).

Let's write code:

```cpp
class Solution {
public:
    int minMoves(vector<int>& nums) {
        int min = INT_MAX;
        int sum = 0;
        size_t n = nums.size();

        for (size_t i = 0; i < n; ++i)
        {
            sum += nums[i];
            if (nums[i] < min)
            {
                min = nums[i];
            }
        }

        return sum - n * min;
    }
};
```

Happy coding :)
