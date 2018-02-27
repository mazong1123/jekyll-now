---
layout: post
title: The trap of Test-Path in PowerShell
date: 2018-02-27 20:11
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Test-Path for unc path return false even if the path exists

I've met this issue when preparing a product environment. After 3 hours investigation, I found that I'm running `Test-Path` from a mapped path: `SQLSERVER:>`. 

The conclusion is, if you running `Test-Path \\network-folder\folder` under a `New-PSDrive` mapped path, you need to be very careful. If the mapped path is created by a `Credential` provider (e.g. `New-PSDrive -Name SQLSERVER -Provider <a-credential-type-provider> -Root SQLSERVER`), `Test-Path \\network-folder\folder` will always return false. You have to user `Test-Path filesystem::\\network-folder\folder` instead.