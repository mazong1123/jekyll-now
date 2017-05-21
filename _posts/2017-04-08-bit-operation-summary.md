---
layout: post
title: Bit operation summary [In Progress]
date: 2017-04-08 19:22
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Bit operation summary

## Remove the right most 1 bit

```cpp
n & (n - 1)
```

Example:
101 & 100 => 100
110 & 101 => 100

## Gray code

LeetCode link: https://leetcode.com/problems/gray-code/#/description

The gray code is a binary numeral system where two successive values differ in only one bit.

Given a non-negative integer n representing the total number of bits in the code, print the sequence of gray code. A gray code sequence must begin with 0.

For example, given n = 2, return [0,1,3,2]. Its gray code sequence is:

00 - 0
01 - 1
11 - 3
10 - 2

Wiki: https://en.wikipedia.org/wiki/Gray_code

> The binary-reflected Gray code list for n bits can be generated recursively from the list for n − 1 bits by reflecting the list (i.e. listing the entries in reverse order), concatenating the original list with the reversed list, prefixing the entries in the original list with a binary 0, and then prefixing the entries in the reflected list with a binary 1.[6] For example, generating the n = 3 list from the n = 2 list:

> 2-bit list:	00, 01, 11, 10	 
Reflected:	 	10, 11, 01, 00
Prefix old entries with 0:	000, 001, 011, 010,	 
Prefix new entries with 1:	 	110, 111, 101, 100
Concatenated:	000, 001, 011, 010,	110, 111, 101, 100

```cpp
class Solution {
public:
    vector<int> grayCode(int n) {
        if (n == 0)
        {
            return vector<int>(1, 0);
        }

        vector<int> p = grayCode(n - 1);
        vector<int> r(p.begin(), p.end());

        int f = pow(2, n - 1);
        for (int i = p.size() - 1; i >= 0; --i)
        {
            r.push_back(p[i] + f);
        }

        return r;
    }
};
```

```cpp
/*
 * This function converts an unsigned binary
 * number to reflected binary Gray code.
 *
 * The operator >> is shift right. The operator ^ is exclusive or.
 */
unsigned int binaryToGray(unsigned int num)
{
    return num ^ (num >> 1);
}
```
