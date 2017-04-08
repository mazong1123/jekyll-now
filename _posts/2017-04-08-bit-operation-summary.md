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
