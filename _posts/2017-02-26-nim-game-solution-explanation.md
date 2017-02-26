---
layout: post
title: Nim game solution and explanation
date: 2017-02-26 11:36
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---
Below is a description of the Nim game:


You are playing the following Nim Game with your friend: There is a heap of stones on the table, each time one of you take turns to remove 1 to 3 stones. The one who removes the last stone will be the winner. You will take the first turn to remove the stones.

Both of you are very clever and have optimal strategies for the game. Write a function to determine whether you can win the game given the number of stones in the heap.

For example, if there are 4 stones in the heap, then you will never win the game: no matter 1, 2, or 3 stones you remove, the last stone will always be removed by your friend.


The problem can be found on [LeetCode](https://leetcode.com/problems/nim-game)

Let's look at the basic case. If 1 or 2 or 3 stones left and it's my turn, I'm gonna win.

So how to guarantee this "win" status? That must be 4 stones left when it's my friend's turn. Because 4 - 1 = 3, 4 - 2 = 2, 4 - 3 = 1. 

How to give the "lose" status to my friend? That must be 5 or 6 or 7 stones left when it's my turn. Becuase we have the chance to make the count of left stones be 4: 5 - 1 = 4, 6 - 2 = 4, 7 - 3 = 4. Then You may find 5 = 1 + 4, 6 = 2 + 4, 7 = 3 + 4. Wow, 1, 2, 3 are our basic cases.

Let's go on. How to guarantee the "win" status of "5 or 6 or 7 stones left when it's my turn"? We should give the "lose" status to my friend that is "8 stones left when it's my friend's turn".Because 8 - 1 = 7, 8 - 2 = 6, 8 - 3 = 5. Again, how to give the "lose" status to my friend? We must be at status of 9, 10, 11 stones left when it's my turn. Because then we have the chance to make the count of left stones be 8: 9 - 1 = 8, 10 - 2 = 8, 11 - 3 = 8. We can find 9 = 1 + 8 = (1 + 4) + 4 = 5 + 4, 10 = 2 + 8 = (2 + 4) + 4 = 6 + 4, 11 = (3 + 4) + 4 = 7 + 4. The formula is now going out: T(n) = T(n - 1) + 4. T(n) is the left stones of the n times of my turn. The basic case T(0) = {1, 2, 3}.

Now we can write a solution like: Test if the number is one of T(n). If yes, we can win. Otherwise we'd lose. Actually it takes O(n) time complexity.

The above solution is really slow to this problem and will lead to a "Time Limit Exceeded" complain.How can we optimize it? Let's expand the formula T(n) = T(n - 1) + 4:

T(n) = {(((1 + 4) + 4) + 4 ..., ((2 + 4) +4) + 4 ..., ((3 + 4) +4) + 4 ...}
     = {1 + 4*n, 2 + 4*n, 3 + 4*n}

Can you get the idea? If T(n) % 4 is 1 or 2 or 3, we can win the game. And beacuse the mod value can only be 0, 1, 2 or 3, that implies if T(n) % 4 != 0, we'll win the game. Now we get the one liner:

```cpp
class Solution {
public:
    bool canWinNim(int n) {
        return n % 4 != 0;
    }
};
```

Happy coding :)