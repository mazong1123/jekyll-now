---
layout: post
title: 501 server cannot accept argument
date: 2017-05-09 16:19
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

# Issue of FTP connection from Linux to Windows Server

If you issue a `ftp` command to connect to a FTP service hosted on Windows Server, you will receive an error:

```
501 server cannot accept argument
```
The `501` error implies that some of your action send to FTP server is not supported. See [this article](https://support.microsoft.com/en-us/help/318380/description-of-microsoft-internet-information-services-iis-5.0-and-6.0-status-codes)

```
501 - Header values specify a configuration that is not implemented.
```

# Solution

The greatest chance of seeing this issue is that your FTP client trying to send `PORT` command to the server, which means using `Active` mode. And apparently, your client side firewall does not allow this.

The solution is to use passive mode, obviously:

```
ftp -p
```

That's it. :)

# References:

https://forums.iis.net/t/1157854.aspx

https://support.microsoft.com/en-us/help/318380/description-of-microsoft-internet-information-services-iis-5.0-and-6.0-status-codes
